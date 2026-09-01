# rusty_agent_gateway

> **Archived — merged into [Rusty Mill](https://github.com/Rusty-Mill/rusty_mill).** This crate now lives at [`crates/rusty_agent_gateway`](https://github.com/Rusty-Mill/rusty_mill/tree/main/crates/rusty_agent_gateway) in the Rusty Mill monorepo, which is where active development, issues, and pull requests happen now. This standalone repo is kept for historical reference only.

A Rust implementation of the [agentgateway] data plane, built to be a drop-in
for its configuration file: an existing `config.yaml` should parse and serve
unmodified.

This is v0. It covers the shared foundation — configuration, listeners, route
matching, policies — plus the **MCP gateway**: several upstream MCP servers
federated into one endpoint, with tool-level filtering and authorization. The
A2A and LLM pillars are not built yet; see [Scope](#scope).

```bash
cargo run -p agentgateway -- --file examples/mcp-federation.yaml
```

## What it does

```yaml
binds:
  - port: 3000
    listeners:
      - routes:
          - matches:
              - path:
                  pathPrefix: /mcp
            policies:
              cors:
                allowOrigins: ["*"]
                exposeHeaders: ["Mcp-Session-Id"]
            backends:
              - mcp:
                  targets:
                    - name: everything
                      stdio:
                        cmd: npx
                        args: ["@modelcontextprotocol/server-everything"]
                    - name: fetch
                      stdio:
                        cmd: uvx
                        args: ["mcp-server-fetch"]
```

One MCP endpoint on `:3000/mcp`, backed by two subprocess servers. Their tools
are unioned and qualified by target, so both can export `search` without
colliding:

```
$ curl … -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}'
everything_echo, everything_add, …, fetch_fetch, …
```

A call to `fetch_fetch` is routed to the `fetch` target and forwarded with the
tool's own name.

## Architecture

| Crate | Responsibility |
| --- | --- |
| `agentgateway-config` | The configuration model. Wire-compatible with agentgateway's local config. |
| `agentgateway-core` | Route matching (Gateway API precedence), hostname patterns, CORS. |
| `agentgateway-auth` | The `jwtAuth` and `extAuthz` policies. |
| `agentgateway-mcp` | MCP federation: targets, tools, prompts, resources, gates, rules, guardrails, spans. |
| `agentgateway-proxy` | HTTP reverse proxying for `host` backends. |
| `agentgateway-tls` | TLS termination, over [`rusty_tls`][rusty_tls]. |
| `agentgateway-llm` | The LLM gateway: an OpenAI-compatible front end over providers. |
| `agentgateway-a2a` | A2A method gating and agent-card discovery, over [`rusty_a2a`][rusty_a2a]. |
| `agentgateway` | The binary: data plane assembly, sockets, graceful shutdown. |

Three decisions are worth knowing up front, because they shape everything else.

**Unknown config fields are accepted, not rejected.** Upstream ships fields
faster than we implement them, and refusing to boot on one we do not support
yet would defeat the point of being a drop-in. `Config::lint` reports what
parsed but is not acted on, and the binary logs each finding at startup — so
the tolerance stays visible rather than silent. A policy that looks enforced
but is not is worse than one that fails loudly, because it looks like security.

**Route precedence follows Gateway API, not file order.** An exact path beats a
prefix, a longer prefix beats a shorter one, then method, header and query
predicates break ties. Prefix matching is segment-aware: `/admin` does not
capture `/admin-public`.

**Tool gates are enforced on `tools/call`, not just `tools/list`.** Nothing
stops a client from calling a name it was never shown. A tool hidden from the
catalogue but still callable is worse than one that was never hidden, because
the operator believes it is gone. There is a test for exactly this.

### Federated names and URIs

Two servers behind one endpoint may both export `search`, so names are
qualified: `github_search`, `jira_search`. Prompts take the same treatment.

Resolving that back is sharper than it looks. Splitting on the first `_` breaks
the moment a target is called `code_search` — its `index` tool federates to
`code_search_index` and splits back to target `code`, tool `search_index`,
routing the call to the wrong server. So resolution matches against the known
target names, longest first. Genuinely ambiguous setups still exist, and
`ToolNamer::collisions` reports them at startup rather than letting the gateway
silently pick one.

Resources cannot: they are identified by URI, and `alpha_memo:insights` is not
a URI any client would accept. Upstream's answer, which this follows, is to
widen the scheme instead — `memo:insights` on target `alpha` becomes
`alpha+memo:insights`, still a well-formed URI because `+` is legal in a
scheme. A URI with no scheme is left alone; there is nothing to widen, and
inventing one would change what the client asked for. Contents come back from
the upstream carrying its own URIs, so they are re-qualified on the way out —
otherwise no client could read the URI back to us.

Set `nameMode: passthrough` on the backend to expose names and URIs unchanged;
a collision is then reported as a startup warning.

Prompt and resource capabilities are advertised only when some target actually
has them, read from what each server sent in its handshake. Claiming prompts
the federation cannot serve would have clients call `prompts/list` and be told
the method does not exist. A target that never advertised prompts is not asked
for them either — a missing capability is not a fault, and a mixed federation is
the normal case.

### Degraded operation

A target that fails to come up does not take the gateway down. Five targets
behind one endpoint means five things that can be broken at any moment, and
refusing to serve the four healthy ones because the fifth is restarting is not
the trade a gateway should make. Failures are logged at startup and reported by
`Federation::degraded`; only losing *every* target is fatal.

## Scope

Implemented and tested:

- `binds` / `listeners` / `routes` / `backends` / `policies`
- Route matching: path (exact, segment-aware prefix, regex), method, headers,
  query — with percent-decoding, and Gateway API precedence
- Hostname matching on listeners and routes, including single-label wildcards
- `mcp` backends: `stdio` and Streamable HTTP (`mcp:`) targets, federating
  tools, prompts and resources with name and URI qualification, `via` egress,
  per-target `filters`, route-level `mcpAuthorization` — both the `allowTools`/`denyTools`
  lists and upstream's CEL `rules`, see [Authorization](#authorization)
- `mcpGuardrails`: external MCP policy processors over gRPC, able to rewrite as
  well as refuse — see [Guardrails](#guardrails)
- `host` and `service` backends: HTTP reverse proxying with weighted load
  balancing, `urlRewrite`, header modifiers and `backendAuth`, over literal
  addresses or a written-down service inventory — see [Proxying](#proxying)
- `dynamic` backends: a forward proxy whose upstream comes from the request —
  see [The client picks the upstream](#the-client-picks-the-upstream)
- `retry` with backoff, and `localRateLimit` token buckets counting either
  requests or LLM tokens — see
  [Retries and rate limits](#retries-and-rate-limits)
- `ai` backends: an OpenAI-compatible API over OpenAI, Anthropic and Gemini,
  streaming and tool calling included on all three, with the `ai` policy's
  `modelAliases`, `prompts`, `defaults`, `overrides`, `promptCaching`, and
  `promptGuard`'s `regex`, `webhook` and `openAIModeration` rules — see
  [The LLM gateway](#the-llm-gateway)
- `a2a` policies: JSON-RPC method gating and a merged agent card — see
  [Agent-to-agent](#agent-to-agent)
- TLS termination with ALPN (`h2` and `http/1.1`), one certificate per
  hostname, and `protocol: TLS` passthrough that forwards connections without
  decrypting them — see [TLS](#tls)
- CORS, including preflight answered at the gateway
- `jwtAuth`: JWKS-backed JWT validation (`url:` or `file:`), issuer and audience
  binding, RFC 6750 `WWW-Authenticate` challenges
- `extAuthz`: an external authorization service consulted before a request is
  served, failing closed — see [External authorization](#external-authorization)
- `timeout`: both `requestTimeout` and `backendRequestTimeout` — see
  [Timeouts](#timeouts), because they bound different things and only one of
  them bounds a tool call
- OpenTelemetry: OTLP traces and metrics, a span per MCP request, and metrics
  labelled by method and tool name
- Process-wide load shedding: a concurrency bound answered with `503` and
  `Retry-After`
- Graceful shutdown on SIGINT/SIGTERM

Parses but is **not** enforced — reported by `--check` and at startup:

- Whatever upstream's `ai` policy grows next: unknown sub-policies are named
  one at a time rather than as the whole of `ai` — see
  [What the `ai` policy does not do yet](#what-the-ai-policy-does-not-do-yet).
  Also `promptGuard.response[].openAIModeration`, which upstream's schema does
  not accept at all: moderation classifies a prompt
- `urlRewrite.authority` where a route has more than one `mcp:` target, and any
  rewrite aimed only at `stdio` targets or using `path.prefix` without exactly
  one `pathPrefix` match — reported by `--check`. Path rewrites apply across a
  whole federation; see
  [Reaching the upstream request](#reaching-the-upstream-request)
- The mesh-only halves of the service inventory — `vips`, `waypoint`,
  `locality`, `subjectAltNames`, `loadBalancer`. They describe a mesh this
  gateway is not part of; what determines routing is implemented. See
  [Naming a service instead of an
  address](#naming-a-service-instead-of-an-address)

Not supported at all:

- The deprecated 2024-11-05 HTTP+SSE target transport (`sse:`). `rmcp` 3.1 has
  no client for it; point the target at the server's Streamable HTTP endpoint
  with `mcp:` instead. Configuring one is a startup error, not a silent skip.
- The `vertex` and `bedrock` providers. Both sign with cloud credentials rather
  than an API key, which is a different kind of work from a translation. They
  parse, and naming one is a startup error for the same reason `sse:` is: a
  provider that loads and never answers is worse than one that refuses to load.
  An OpenAI-compatible endpoint in front of them works today via `openAI` with
  a `hostOverride`.

## Authentication

A route with a `jwtAuth` policy requires a bearer token:

```yaml
policies:
  jwtAuth:
    issuer: https://auth.example.com
    audiences: ["https://gateway.example.com/mcp"]
    jwks:
      url: https://auth.example.com/.well-known/jwks.json
```

Three details are worth knowing, each with a test named after it.

**The audience check is ours, and it is the point.** [`rusty_mcp`][rusty_mcp]'s
`JwtValidator` reads `aud` and deliberately checks nothing, because upstream's
own layer binds it to a single canonical resource URI while our `audiences` is a
list. Skipping it would be the confused-deputy hole: a caller replays a token
minted for some other service and borrows this gateway's privileges. An empty
`audiences` accepts any audience — a documented opt-out for deployments that
bind audience upstream, not an oversight.

**Authentication runs after the CORS preflight branch.** Browsers never send
`Authorization` on a preflight, so requiring a token there would make every
cross-origin call fail before the real request was ever sent. Rejections still
carry CORS headers, because a 401 the browser cannot read is a 401 nobody can
act on.

**A validator outage is a 503, never a 401.** If the JWKS endpoint is
unreachable the client's token may be perfectly good; answering 401 would send a
user through a login that fixes nothing and disguise an outage as an auth
problem.

A `file:` JWKS is read once at startup, so a missing, malformed or empty key set
stops the gateway booting instead of turning every request into a runtime error
that reads like a client problem. Rotating that file needs a restart — which is
what the `url:` form is for.

## External authorization

An `extAuthz` policy asks an authorization service about each request before it
is served:

```yaml
policies:
  extAuthz:
    target: http://authz.internal:9000
    timeout: 250ms
    includeHeaders: [authorization]
    allowedUpstreamHeaders: [x-user-id]
```

The call carries the request's original method and path, so the authorizer can
route on what is being authorized instead of reading it back out of a header. A
2xx allows the request; anything else denies it, and the authorizer's own
status, body and `WWW-Authenticate` are returned to the caller — an authorizer
answering `403 {"reason": "not in group"}` is telling the caller something a
generic "forbidden" would throw away.

**It fails closed.** When the authorizer cannot be reached the request is
denied, because an authorization service that is down must not become an open
door. `failOpen: true` reverses that for deployments that would rather serve
than stall, but it has to be asked for. The refusal is a **503, not a 403**:
nothing decided the request was forbidden, and saying "forbidden" would send
someone to check their permissions when the real problem is a service being
down.

**Both header lists are allow-lists, and the outbound one matters more.**
`includeHeaders` bounds what the authorizer sees, keeping cookies and payloads
away from a service that has no need for them. `allowedUpstreamHeaders` bounds
what the authorizer may set on the request that continues upstream — without
it, an authorizer could write any header the upstream trusts (`x-user-id`,
`x-is-admin`), which turns an authorization service into an impersonation
service. Both default to empty, which allows nothing rather than everything.

The default budget is 250ms. This call sits in front of every request on the
route, so a slow authorizer is a slow gateway; exceeding the budget is an
unreachable authorizer and takes the fail-closed path.

`extAuthz` runs after `jwtAuth`, so an authorizer configured with
`includeHeaders: [authorization]` sees a token the gateway has already
validated.

### Showing the authorizer the body

Some decisions need the payload — which tool a JSON-RPC call names, which model
a completion asks for — and none of that is in a header.

```yaml
policies:
  extAuthz:
    target: http://authz.internal:9000
    includeBody: 4096
```

Buffering an arbitrary upload to show someone would turn the gateway into a
memory limit, so `includeBody` caps it. What happens at the cap is the
interesting part: **a body over the limit is refused with `413`, not
truncated.** Sending the first N bytes would ask the authorizer to decide on a
fragment, and a fragment of JSON does not parse — so it would answer about
something that was never the request. That is a worse failure than a `413` and
a much quieter one. It is the same instinct as failing closed on an unreachable
authorizer: no decision is not the same as yes.

The bound is inclusive, so a config sized to the largest expected payload does
not refuse it. Nothing is read at all without `includeBody`, so a route that
does not want this does not pay for it.

The body's `Content-Type` travels with it whether or not `includeHeaders` names
it, because a payload whose format the authorizer has to guess is not much use.

Reading the body does not consume it: what the authorizer saw is what continues
upstream, on every backend kind — a proxied `host` request, an `a2a` call, an
`ai` completion the gateway translates.

## Authorization

An `mcpAuthorization` policy decides what a caller on the route may reach.
Two forms, and they compose:

```yaml
policies:
  mcpAuthorization:
    # Regexes over the federated tool name. Tools only.
    denyTools: ["_delete$"]
    # CEL expressions over the call and the caller.
    rules:
      - 'mcp.tool.name == "echo"'
      - 'jwt.sub == "test-user" && mcp.tool.name == "get-sum"'
      - 'mcp.prompt.name == "summarize"'
      - 'mcp.resource.name.startsWith("memo:")'
      - require: 'jwt.iss == "https://auth.example.com"'
```

`allowTools`/`denyTools` are name matching and nothing else, and — as the name
says — they are about tools. `rules` cover **tools, prompts and resources**, and
are what you need when the answer depends on *who* is asking: `jwt` is the
verified token's claims, so a rule can grant one thing to one subject and
another to everyone.

Every gate is enforced on the call, not only on the listing — `tools/call`,
`prompts/get` and `resources/read` each re-check. Filtering the catalogue alone
is security theatre: nothing stops a client asking for a name it was never
shown, and something hidden from the listing but still reachable is worse than
something that was never hidden, because the operator believes it is gone.

### One subject is bound per call

Exactly one of `mcp.tool`, `mcp.prompt` and `mcp.resource` exists for any one
call. On a `prompts/get`, `mcp.tool.name` does not resolve, so
`mcp.tool.name == "echo"` reads as **false** — not as "this rule isn't about
prompts, skip it".

That has a consequence worth stating plainly, because it is the one that
catches people:

> **A rule set written entirely as tool `allow` rules refuses every prompt and
> resource.** It is an allow-list, and nothing in the prompt or resource space
> can satisfy it.

This is the safe direction — a policy that had never heard of prompts should not
wave them through — and it is upstream's behaviour. But it does mean that adding
prompts to a target behind an existing tool allow-list takes an explicit rule.
A pure `deny` set behaves the opposite way: it names what is refused, so a
prompt it does not name survives.

### A rule sees the unqualified name

`mcp.tool.name` is the tool's own name on its target, and `mcp.tool.target` is
the target — the pair *before* federation joins them. A tool federated as
`everything_echo` is `mcp.tool.name == "echo"` with `mcp.tool.target ==
"everything"`, so a rule written against `everything_echo` never fires. Prompts
work identically, and a resource's `mcp.resource.name` is the target's own URI:
`memo:insights`, not the federated `alpha+memo:insights`.

(`mcp.resource.name`, not `.uri` — upstream names the field for the shape it
shares with tools and prompts rather than for what a resource calls its
identifier.)

That is upstream's split, and it lets a rule name something without knowing what
the gateway will prefix it with. It is also the opposite of
`allowTools`/`denyTools`, which match the federated name — which is what lets
one route ban a tool on one target while leaving the same name on another
alone. Both readings are right for their own form; the difference is worth
knowing because getting it wrong makes a policy silently not apply.

### Precedence

1. No rules at all permits, so adding an empty `rules:` list does not take the
   route offline.
2. Any `deny` that holds refuses, ahead of everything else.
3. Every `require` must hold.
4. Any `allow` that holds permits.
5. Otherwise: permitted only if there were no `allow` rules. A set of pure
   `deny` rules is a deny-list, so what it does not name survives; the moment
   one `allow` exists the set becomes an allow-list.

A bare string is an `allow`, which is how upstream's examples are written.

### Prefer `require` to `deny`

An expression that cannot be evaluated — no `jwt` on a route without `jwtAuth`,
a claim that is not there, a type mismatch — counts as **false**. That is
upstream's behaviour, and it is the safe direction for `allow` and `require`,
which fail towards refusing.

It is the wrong direction for `deny`: a `deny` that errors permits the call.
So `deny: 'jwt.role == "banned"'` on a request with no token lets the request
through, where `require: 'jwt.role != "banned"'` refuses it. Both are
supported, and there is a test for each pinning the difference down.

An expression that does not compile is a startup failure, not a skipped rule —
otherwise a typo in an `allow` silently serves nothing and a typo in a `deny`
silently refuses nothing.

## Guardrails

An `mcpGuardrails` processor is an MCP-aware policy service the gateway
consults over gRPC — Envoy's `ext_authz` shape moved down to the MCP method
layer, with one addition that changes what it is for: **a processor can rewrite
as well as refuse.** Redacting a secret out of a tool result is not something a
yes/no answer can do.

A processor names its address one of three ways, and exactly one:

```yaml
- host: guardrail.internal:9000     # literal
- backend: guard                    # an entry in the top-level `backends:`
- service:                          # an entry in the inventory
    name: guard
    port: 9000
```

`backend:` and `service:` resolve through the same registry a `service` backend
uses — see [Naming a service instead of an
address](#naming-a-service-instead-of-an-address). A `service` can resolve to
several instances, so a processor holds a ring rather than an address and
spreads its calls across them; sending every call to the first would make the
others decoration. A name that does not resolve is a **startup failure**, since
a processor that cannot be reached is a guardrail that is not running.

Naming none of the three, or more than one, is reported by `--check`: the first
is a policy that does not describe a processor, and the second says two
different things where picking either would be a guess.

```yaml
policies:
  mcpGuardrails:
    processors:
      - kind: remote
        host: guardrail.internal:9000
        timeout: 5s
        methods:
          tools/call: full        # both phases
          "prompts/*": full       # a whole namespace
          "*/list": response      # results only
        failureMode: failClosed   # the default
        metadata:
          tenant: 'request.headers["x-tenant"]'
        requestHeaders:
          allowed: [x-tenant]
```

The wire protocol is upstream's `agentgateway.dev.ext_mcp` — `CheckRequest` and
`CheckResponse`, each answering pass, a replacement body, or a refusal.
`proto/ext_mcp.proto` in `agentgateway-mcp` is upstream's schema, unchanged. The
messages are written by hand rather than generated: `tonic-build` would put
`protoc` in everyone's build for eight small messages, and the field numbers are
pinned against encoded bytes in a test rather than against the declarations.

### What is hooked

A processor can hook any of the seven methods this gateway serves:

| Method | Request phase sees | Response phase sees |
| --- | --- | --- |
| `tools/list` | nothing to rewrite | the merged catalogue |
| `tools/call` | `{"name": "echo", …}` | the `CallToolResult` |
| `prompts/list` | nothing to rewrite | the merged listing |
| `prompts/get` | `{"name": "summarize"}` | the `GetPromptResult` |
| `resources/list` | nothing to rewrite | the merged listing |
| `resources/templates/list` | nothing to rewrite | the merged listing |
| `resources/read` | `{"uri": "memo:insights"}` | the `ReadResourceResult` | A processor keyed only on methods this gateway does
not serve — `logging/setLevel`, `completion/complete` — is reported by `--check`
and at startup, because a guardrail that never fires looks exactly like one that
always passes.

The single-target methods — `tools/call`, `prompts/get`, `resources/read` — run
both phases. The `*/list` methods fan out, so their request phase runs once for
the whole client call and carries no params: a processor can refuse there but
has nothing to rewrite, and filtering a catalogue is response-phase work.

Guardrails run **after** `mcpAuthorization`, so a processor is only consulted
about calls the route was otherwise going to serve — a guardrail should not be
billed for traffic already refused. An upstream *error* skips the response phase entirely: there is no result
to inspect, and asking a guardrail to approve a failure is not a question it can
answer.

### The two phases do not see the same names

On the **request** side a processor sees the **unmuxed** identifier — what the
upstream will actually receive. `alpha_echo` arrives as `{"name": "echo"}` with
`service_names: ["alpha"]`, and `alpha+memo:insights` arrives as
`{"uri": "memo:insights"}`.

On the **response** side it sees what the *client* will get, which for
resources means the federated form: a `resources/read` result reaches the
response phase with its contents already re-qualified to
`alpha+memo:insights`, and a `resources/list` carries federated URIs
throughout.

The asymmetry is not an oversight — each phase shows the form that is
actionable at that point — but a filter written against one form will silently
not match the other, so it is worth knowing which side you are on. Tools never
made this visible because a `CallToolResult` carries no names.

A request-phase rewrite that hands back the federated URI it saw on a listing
is unwrapped rather than passed through to fail, since the upstream only knows
its own URIs.

Note also that a rewrite is **not re-authorized**: a processor that rewrites
`{"name": "summarize"}` to `{"name": "leak"}` gets `leak`, even where a rule or
an `allowTools` entry would have refused it. That is upstream's contract —
"the gateway does not re-run other RBAC on the mutated request" — and it means
a processor with rewrite authority is as trusted as the policy itself.

### A chain is a pipeline, not a vote

Processors run in configuration order, and each one sees what the previous one
produced. A redactor followed by a validator should see redacted input. The
first refusal ends the chain — the processors after it are not consulted, and
neither is the upstream.

### Method matching

`methods` is an allow-list from pattern to phase (`off`, `request`, `response`,
`full`); a method matching no key bypasses the processor. When several patterns
match, the most specific wins: an exact name, then a prefix wildcard
(`tools/*`), then a suffix wildcard (`*/list`), then `*`. Within one kind the
longer pattern wins, and remaining ties break alphabetically so resolution never
depends on map ordering. A pattern that can never match — `a*b`, `**` — is
reported rather than accepted.

### Failing closed

A processor that cannot be reached, exceeds its budget (10s by default), or
answers something unparseable **refuses the call**. `failureMode: failOpen`
reverses that but has to be asked for, for the same reason as `extAuthz`: a
policy service that is down must not silently become an open door.

A refusal carries the processor's own reason and code — `permissionDenied`
becomes JSON-RPC `-32001`, `resourceExhausted` `-32003`, `invalid` `-32600`, and
anything else `-32603`. (`-32002` is skipped: `rmcp` already assigns it to
`RESOURCE_NOT_FOUND`.)

### Header and metadata context

`requestHeaders` bounds what the processor is shown. Note the default: an empty
`allowed` forwards **every** header, which is upstream's reading and the
opposite of `extAuthz.includeHeaders`. `disallowed` always wins, and both match
case-insensitively.

`metadata` is a map of CEL expressions evaluated per call and sent as
`metadata_context`, which is how a processor gets context the protocol does not
carry. Three things are in scope:

- `jwt` — the verified token's claims
- `request` — `method` and `headers`
- `mcp` — the subject the call is about, in the same shape
  `mcpAuthorization.rules` uses

```yaml
metadata:
  tenant: 'request.headers["x-tenant"]'
  prompt: 'mcp.prompt.name'
  resource: 'mcp.resource.name'
  who_wants_what: 'jwt.sub + " -> " + mcp.resource.name'
```

`mcp` is **ours, not upstream's**. Upstream evaluates these expressions against
the HTTP request alone and reserves the MCP context for RBAC. Adding it is
additive — no expression that worked before changes meaning, and
`metadata_context` is an opaque struct on the wire — and it is what lets a
processor be handed the prompt or resource it is being asked about without
parsing `mcp_request` itself.

The same one-subject-at-a-time rule applies as in `rules`: exactly one of
`mcp.tool`, `mcp.prompt`, `mcp.resource` exists per call, and it holds the
**unmuxed** identifier — `summarize`, `memo:insights`. A key whose expression
cannot be evaluated is dropped rather than failing the call, so this is safe,
but it has a consequence worth seeing:

```
tools/call     → {method, target, tool}        # target: 'mcp.tool.target'
prompts/get    → {method, prompt}              # `target` vanished
resources/read → {method, resource}
prompts/list   → {method}                      # a fanout has no one subject
```

A `target` key written as `mcp.tool.target` silently disappears on prompt and
resource calls. Write it per kind, or read `service_names`, which is on every
message regardless.

An expression that cannot be evaluated is dropped rather than failing the call:
metadata is context for the processor, not a decision, and a missing claim
should not take a guardrail offline. One that does not *compile* is a startup
failure.

### What a processor sends back

`metadata_context` goes to the processor. `McpRequestResult.metadata` comes
**back** — an arbitrary bag a processor fills in with whatever it decided: the
classification it made, the rule it matched, the tenant it resolved.

Upstream stashes that for downstream CEL filters (`transformation`) to read.
This gateway has none, and a bag nothing can read is worse than no bag at all,
so the values go on the request's telemetry span instead:

```
McpRequestResult.metadata = {classification: "phishing", rule: "r-9b2e"}
                          ↓
span tools/call
  mcpGuardrails.classification = "phishing"
  mcpGuardrails.rule           = "r-9b2e"
```

A decision a guardrail took in-band becomes visible in the trace afterwards,
rather than only in the processor's own logs. Keys are namespaced by the policy
that produced them, so a processor cannot collide with the gateway's own
attributes.

Scalars go on natively, so a trace viewer can filter on them; anything nested
is rendered as JSON, because OpenTelemetry attribute values are scalars and
flattening an object into dotted keys would invent structure the processor did
not ask for. Note that a whole number comes back as a double — protobuf's
`Struct` has one number type, and that is the protocol's choice rather than
ours.

A processor is an external service, so the bag is bounded: at most 32 keys and
1 KiB per value, with the excess dropped or truncated and a warning logged.
Without that, a processor could grow every span the gateway exports. Later
processors in a chain win a key collision, on the grounds that the last thing
to say something about a call is the most informed. A refusal carries no
annotations at all, for the same reason it carries no header changes.

Setting an attribute is a no-op unless `config.tracing` names a collector, so
this costs nothing when tracing is off.

### Reaching the upstream request

The other half of what upstream's metadata bag is for. `requestHeaderModifier`
was consumed only by the `host` proxy path — on a route with an `mcp` backend it
parsed and did nothing. It applies to MCP upstream requests now, and its values
can reference what a guardrail decided:

```yaml
policies:
  mcpGuardrails:
    processors: [...]           # returns metadata {classification: "phishing"}
  requestHeaderModifier:
    set:
      x-classification: "{{mcpGuardrails.classification}}"
      x-rule: "rule={{mcpGuardrails.rule}}"
      x-api-key: "static-value"
```

A processor classifies a call in-band, and the MCP server behind the gateway is
told, without ever speaking to the policy service itself. That is upstream's
`transformation` consumer, in the shape this gateway already has.

`{{...}}` placeholders rather than bare CEL, because a header value is a string
and most of them are literals — requiring a delimiter means adding this cannot
change what an existing static value means. Only `mcpGuardrails.<key>` resolves;
anything else is a **startup failure**, rather than a header that silently never
fires, which would read exactly like a guardrail that never ran.

An unresolved placeholder **drops its header** rather than sending
`{{mcpGuardrails.classification}}` upstream as though it were data. A guardrail
that did not run, or did not set that key, should read as "no classification",
and an absent header says that where a literal template string says something
false. Other headers in the same modifier are unaffected.

The modifier runs **after** a guardrail's `headerMutation`, so route
configuration wins over a processor's runtime decision — the operator's intent
is the one written down, and upstream's ordering says the bag exists so that
*subsequent* filters can read it.

Two smaller things worth knowing. `add` cannot append the way it does on the
HTTP proxy path, because one value per name crosses to the transport; a name
already spoken for is joined into one comma-separated field line instead. And
the startup warm-up — the one listing the gateway makes on its own behalf to
build the name index — carries static values but resolves no templates, since
nothing has classified anything at that point.

Only `mcp:` targets, for the same reason as `headerMutation`: a `stdio` target
speaks over a pipe and has no headers.

### The same modifier on an `ai` or `a2a` route

`requestHeaderModifier` applies to the request that leaves the gateway on
**every** backend kind, not only `host` and `mcp`:

```yaml
policies:
  requestHeaderModifier:
    set: { x-tenant: acme }
    add: { x-scope: models }
    remove: [authorization]
backends:
  - ai:
      provider:
        openAI: {}
```

An `a2a` route already had this, because it dispatches through the same `host`
proxy — including on the path where the policy has to buffer the body to read
the JSON-RPC method out of it. There are tests for both now, since "it already
works" is the claim worth checking rather than assuming.

An `ai` route did not. The request that reaches a model provider is *built* by
the gateway rather than forwarded — a translated body, the provider's own
endpoint, a credential from `backendAuth.key` — so nothing was ever going to
apply a route's modifier to it, and it parsed and did nothing.

It runs **after** the provider's own headers, the same ordering the `host`
proxy uses for `backendAuth`: a route that names a header means it, even one
the gateway put there. That is what makes `remove: [authorization]` say
something useful — this route does not hand a key to the provider — and it is
worth being able to say. `set` on a name the provider took replaces it; `add`
appends a second field line, which here it genuinely can, unlike the MCP path
where one value per name crosses to the transport.

A name or value HTTP cannot represent fails the route at **startup**, next to
the `ai` provider errors, rather than dropping a header on every call where
nobody would see it.

### Response headers, and the one policy that cannot apply

`responseHeaderModifier` applies to whatever the route's backend produced, on
**every** backend kind:

```yaml
policies:
  responseHeaderModifier:
    set: { x-served-by: rusty-agent-gateway }
    remove: [mcp-session-id]
```

It used to live inside the `host` proxy, which meant it reached a proxied
upstream response and nothing else — not an `ai` completion, not an A2A card or
refusal the gateway answers itself, not an MCP response, since none of those go
through the proxy. It is applied where the backends converge instead, so one
description of the policy is true of every route.

On an MCP route there is no upstream HTTP response to modify at all: `rmcp`'s
transport consumes those and a client never sees one, so the only response
worth acting on is the one going back out.

It stays scoped to *backend* responses. A CORS preflight, a JWT challenge and
an `extAuthz` refusal are answered before dispatch and are the gateway's own
rather than the route's payload. CORS is added after the modifier runs for the
same reason — a route cannot accidentally strip the headers that answer a
preflight.

`urlRewrite` replaces parts of the one address the gateway dials. All three of
`authority`, `path.full` and `path.prefix` apply, where there is a single
Streamable HTTP target to be unambiguous about:

```yaml
matches:
  - path: { pathPrefix: /mcp }
policies:
  urlRewrite:
    authority: mcp.internal:8443    # replaces the target's host and port
    path: { prefix: /rpc }          # /mcp/v1 -> /rpc/v1
backends:
  - mcp:
      targets:
        - name: alpha
          mcp: { host: 127.0.0.1, port: 3001, path: /mcp/v1 }
```

A **path** rewrite works across any number of targets. It transforms each
target's own configured path and leaves its host alone, so a federation of
servers that agree on a path layout moves together:

```yaml
policies:
  urlRewrite:
    path: { prefix: /rpc }
backends:
  - mcp:
      targets:
        - name: alpha
          mcp: { host: a.internal, port: 3001, path: /mcp/a }   # -> /rpc/a
        - name: beta
          mcp: { host: b.internal, port: 3001, path: /mcp/b }   # -> /rpc/b
```

An **authority** does not generalise the same way. Pointed at several targets
it would make them all the same server — not a redirect but a collapse, since a
target's address is exactly what distinguishes it from the others. Over a
single target it is a redirect and applies; over several it is reported and the
path half of the same rewrite still runs.

### Collapsing a federation onto one address

The collapse is sometimes exactly what you want — an egress proxy or a mesh
sidecar that all upstream traffic goes through. It has its own spelling, on the
backend that owns the targets rather than on a route policy, so nobody reaches
it while meaning to write a redirect:

```yaml
backends:
  - mcp:
      via: egress.local:8443      # every target dialled here
      targets:
        - name: alpha
          mcp: { host: a.internal, port: 3001, path: /a }
        - name: beta
          mcp: { host: b.internal, port: 3001, path: /b }
```

Each target keeps its own path, which is then the only thing telling them
apart. Two that would end up at the same address *and* path are two connections
to the same endpoint federating the same tools twice, so `--check` reports it:

```
warning: ...mcp.via: targets `alpha` and `beta` would both be dialled at
         `egress.local:8443` port 8443 path `/same`, so they are the same
         endpoint federated twice; only their paths tell them apart once collapsed
```

`via` follows the same port rule as `urlRewrite.authority` — an address naming
no port keeps each target's own — and refuses userinfo for the same reason.
When a config sets both, `via` wins: it is the more specific of the two and the
one that names targets, so the outcome does not depend on which is read first.

**The path a rewrite acts on is the target's own configured path**, not
anything derived from the request. An MCP route terminates the protocol and
dials its session once at startup, long before a request exists, so there is no
per-request path to rewrite — and `prefix` replaces the route's matched prefix
at the head of the configured path instead. That is the same operation the
proxy performs, on the only path this model has, using the same implementation
so the two cannot drift.

Reusing it brings one behaviour worth knowing: when the configured path does
**not** start with the matched prefix there is nothing to strip, so the
replacement lands in front of the whole path. Under a `/mcp` → `/rpc` rewrite,
a target at `/other` is dialled at `/rpc/other`. A `prefix` that behaved
differently here than on a `host` route would be a difference nobody could see
coming from the config, so it behaves the same.

An authority that names **no port keeps the target's own**. The target names a
port explicitly and the override did not, so dropping to 80 would break a
config that only meant to move hosts. Write `host:port` to move both.

An authority carrying userinfo is a **startup failure**:

```
route #0.urlRewrite.authority: `admin:hunter2@10.0.0.1:8080` is not a valid
authority: userinfo does not belong in an upstream address, use `backendAuth`
```

It is legal syntax, but a credential in an upstream URI hides somewhere nobody
thinks to look and is sent on every request. `backendAuth` is where one
belongs.

What `--check` reports rather than quietly ignoring:

| | Why not |
| --- | --- |
| `authority`, several targets | It would point them all at one server rather than redirecting them. |
| A rewrite where no target is `mcp:` | A pipe has no address. |
| `path.prefix`, not exactly one `pathPrefix` match | Which prefix a request matched is not knowable when the session is dialled. Use `full` to set the path outright. |

The last of those covers an `ai` route too: it resolves one endpoint at startup,
before any request exists, so it faces the same question. A `host` route
rewrites per request and knows what that request matched, so it is exempt.

### `urlRewrite` on an `ai` or `a2a` route

Same policy, same three fields, on the remaining two backend kinds.

An **`a2a`** route dispatches through the `host` proxy, so it already rewrote
per request, on both the ordinary path and the one where the policy has to
buffer the body to read the JSON-RPC method out of it. Both are tested now
rather than assumed.

What is new there is **discovery**. Agent cards are fetched at startup, and that
fetch now follows a rewritten `authority`:

```yaml
policies:
  urlRewrite:
    authority: egress.local:8443   # calls *and* card discovery go here
  a2a:
    agentCard: { url: "https://gateway.example.com/a2a" }
backends:
  - host: "agent-a:9000"
```

A gateway that fetched cards from an address it never sends traffic to would
serve a card describing the wrong agents — and behind an egress proxy that is
the only route to them, no card at all. Every backend behind one rewritten
authority is the same address, so it is fetched once rather than once per
backend, which would otherwise merge an agent with itself.

A **path** rewrite deliberately does not follow: the well-known path is the A2A
spec's, not the route's, and asking an agent for its card somewhere else finds
nothing.

An **`ai`** route is the one that did nothing before. Its request is built
rather than forwarded, so the address being rewritten is the *provider's
endpoint*:

```yaml
policies:
  urlRewrite:
    authority: acme.openai.azure.com
    path:
      full: /openai/deployments/gpt4o/chat/completions?api-version=2024-02-01
backends:
  - ai:
      provider:
        openAI: {}
```

**The path acted on is the provider's, not the client's.** A client's request
path never reaches a provider — the endpoint's path is the provider's API,
`/v1/chat/completions` or `/v1/messages`. So `full` replaces that, which is how
an Azure-style or gateway-mounted deployment is reached, and `prefix`
transforms it against the route's own matched prefix, exactly as an `mcp`
target's configured path is transformed:

```yaml
matches:
  - path: { pathPrefix: /v1 }
policies:
  urlRewrite:
    path: { prefix: /openai/v1 }    # /v1/chat/completions -> /openai/v1/chat/completions
```

`authority` and `hostOverride` **compose rather than competing**, because they
are not the same operation: `hostOverride` is a base URL and carries the
scheme, while `authority` replaces only host and port. Setting both means "this
route talks to a self-hosted compatible endpoint over http, and its egress goes
through that address" — the `host` proxy's arrangement of a backend address
plus a rewrite, in the shape an `ai` backend has. That is deliberately not the
rule `mcp` follows for `via` versus `urlRewrite.authority`, where one wins:
those two *are* the same operation spelled twice, so one had to.

A rewrite that cannot be applied — an endpoint that is not an absolute URL —
is a **startup failure**, not a silent no-op. The config says the gateway
should be dialling somewhere else; serving traffic to the original address
instead is the outcome nobody asked for.

`hostOverride` carrying userinfo is a startup failure too, for the same reason
`urlRewrite.authority` always was: a credential there is sent on every request
from a place nobody reads, and would be logged with the resolved endpoint
besides.

### The span itself

There was not one before this. `tracing` fields have to be declared when a span
is opened, so arbitrary processor-supplied keys cannot be recorded as fields at
all — they go on as OpenTelemetry attributes, which need an OpenTelemetry span
to go on. It is worth having on its own: a `tools/call` that took four seconds
was not visible anywhere otherwise.

The span carries the client's W3C trace context when `_meta` propagated one, so
a request joins the caller's trace rather than starting its own.

It is deliberately **not** entered. The OpenTelemetry layer times a span from
creation to close rather than from enter to exit, so the duration is right
either way — and `Span::enter` returns a thread-bound guard, so holding one
across an `.await` in a server that multiplexes tasks onto a thread pool would
attribute another request's events to this span. The trade is that log lines
inside a handler are not nested under it.

### `headerMutation`

A processor's request-phase answer can also change the headers of the upstream
HTTP request that carries the call — which is how a guardrail passes a resolved
identity to the MCP server behind it:

```
McpRequestResult.header_mutation = { set: [{key: "x-user-id", value: "u-42"}] }
```

It is a **per-call** change, not a connection one. The connection to a target is
dialled once at startup and shared, so the change rides in the request's
extensions, which `rmcp` carries in memory from the peer down to the transport.
The handshake, which happened before any processor was consulted, never carries
it.

Only `mcp:` targets have headers to change. A `stdio` target speaks over a pipe,
so a mutation aimed at one is logged and dropped rather than failing the call.
For `tools/list`, which fans out, the change applies to every target's request —
there is one client call and several upstream ones, and singling one out would
be arbitrary.

Changes accumulate across the chain: a later processor setting a name an earlier
one set wins, a later `remove` cancels an earlier `set`, and vice versa. Within
one processor's answer, repeated `set` entries for one name are joined with
`", "` — the protocol says they form a list replacing the header, and a single
comma-separated field line is how HTTP spells that. A name or value HTTP cannot
represent is skipped with a warning rather than failing the call. A refusal
carries no header changes at all: a request that never happens should leave no
trace on the one that replaces it.

## Proxying

A `host` backend forwards bytes rather than terminating a protocol, which makes
it where the policies that only make sense for a proxy finally do something:

```yaml
- name: api
  matches:
    - path:
        pathPrefix: /api
  policies:
    urlRewrite:
      path:
        prefix: /v1          # /api/things -> /v1/things
    requestHeaderModifier:
      set: {x-gateway: rusty}
    backendAuth:
      key: backend-secret
  backends:
    - host: "10.0.0.1:8080"
      weight: 3
    - host: "10.0.0.2:8080"
      weight: 1
```

Weighted selection is deterministic round-robin over a weighted ring, not
random choice: randomness only reaches the configured ratio in expectation, so
a low-traffic route can sit lopsided for a long time. Weight `0` drains a
backend without deleting its configuration; every weight being `0` is a startup
error, since the route could never send traffic anywhere.

Hop-by-hop headers are stripped in both directions — **including the headers
the `Connection` header names**, which is the half that is easy to miss:
`Connection: x-custom` makes `x-custom` hop-by-hop for that message, and a
proxy that only removes the fixed RFC list happily forwards it.

The client address is appended to `X-Forwarded-For` rather than replacing it,
because overwriting the chain erases every proxy before us. The upstream's
`Host` is rewritten to name the upstream, or a name-based virtual host serves
the wrong site. `backendAuth: key` replaces the client's own `Authorization`
rather than adding to it, so a client cannot smuggle its credential to a
backend that should see only ours.

An unreachable upstream is a `502`, not a `500`: the gateway is fine, the
upstream is not, and conflating the two sends people to debug the wrong
process. Unlike MCP, a proxied response is not produced until the upstream
answers, so `backendRequestTimeout` genuinely bounds the wait here.

A route mixing `host` with a backend kind this build cannot serve is refused
rather than served: silently dropping the unsupported share onto the hosts
would send traffic somewhere the operator never asked for.

### Naming a service instead of an address

Upstream learns services and their endpoints from a control plane, and its
local configuration file can carry that same inventory written down by hand.
That is the half a file-driven gateway can serve, and it is what `service`
backends resolve against:

```yaml
binds:
  - port: 3000
    listeners:
      - routes:
          - backends:
              - service:
                  name: default/echo.default.svc.cluster.local
                  port: 80

services:
  - name: echo
    namespace: default
    hostname: echo.default.svc.cluster.local
    ports:
      80: 8080              # asked for : listened on

workloads:
  - name: echo-1
    namespace: default
    workloadIps: ["10.0.0.1"]
    services:
      "default/echo.default.svc.cluster.local": {}
  - name: echo-2
    namespace: default
    workloadIps: ["10.0.0.2"]
    services:
      "default/echo.default.svc.cluster.local": {}
```

The two lists are **joined rather than nested**, exactly as a control plane
sends them: a service declares a hostname and its ports, and a workload claims
membership. Nothing in a service names its endpoints, which is what lets
instances come and go without rewriting it.

A backend may name a service three ways — the full `namespace/hostname`, the
hostname, or the short name — most specific first. A short name two namespaces
share is reported as **ambiguous** rather than resolved by sort order.

Ports are mapped twice: the service maps the port a caller asks for onto the
port a workload listens on, and a workload's own entry may map it again,
because one instance in a set can differ from the rest. The workload's answer
wins where it has one.

**Weights are split, not repeated.** A route sending half its traffic to a
service and half to a host means half, however many instances the service has —
so every backend's weight is scaled by the least common multiple of the
endpoint counts and divided by its own. Giving each instance the backend's
weight would make a three-instance service take three quarters. Weight `0`
still drains, since scaling must not lift a drained backend back in.

An unhealthy workload is left out of the set. A service nothing healthy backs
is a **startup failure**: a route pointed at one would answer every call with
an error, and in a file-driven inventory nothing is going to come along and
fill it in. So is a service the inventory does not hold, and the error names
what it does hold.

Everything else a control plane sends — `vips`, `waypoint`, `locality`,
`subjectAltNames`, `loadBalancer` — describes a mesh this gateway is not part
of. It parses so an upstream file loads, and `--check` names it.

### The client picks the upstream

A `dynamic` backend has no address in the configuration at all. The route
becomes a forward proxy:

```yaml
backends:
  - dynamic: {}                                     # the request's own authority
  - dynamic:
      target: 'request.headers["x-upstream"]'       # or CEL over the request
```

This is worth stating plainly, because it is the whole feature and also its
whole risk. With no `target`, the route dials whatever authority the request
carries — so **anyone who can reach the listener can make the gateway open a
connection anywhere the gateway can**, from the gateway's own network position.
That is a forward proxy, which is a legitimate thing to run and a catastrophic
thing to run open. A route carrying one is logged at startup and reported by
`--check`, and both say to put authentication and authorization in front of it.

A `target` expression moves the decision; it does not remove it. The expression
reads the client's request too, so pointing it at a header only helps if a hop
the operator trusts is the one setting that header and a policy strips whatever
the client sent. It may read `request.headers`, `request.method`,
`request.path` and `request.authority`, and must produce a `host:port` string —
anything else, or a request naming nowhere at all, is a **400** rather than a
guess. An expression that does not compile is a startup failure, since falling
back to the request's authority would quietly choose a different upstream.

Two things follow from having no fixed address. A `dynamic` backend cannot be
weighted against one that has one — a share of "wherever the client asked for"
is not a destination anyone can reason about — so it must be the only backend
on its route. And `urlRewrite.authority` is ignored with a warning: a forced
authority would mean the request never chooses one, which is the opposite of
the point. A path rewrite still applies.

Retries go back to the same place, because there is no second instance to fail
over to and inventing one would mean dialling somewhere the request did not
name.

## Agent-to-agent

```yaml
policies:
  a2a:
    denyMethods: ["^tasks/cancel$"]
    agentCard:
      url: "https://gateway.example.com/a2a"   # what clients should call
backends:
  - host: "agent-a:9000"
  - host: "agent-b:9000"
```

An `a2a` policy marks a route as carrying [Agent2Agent] traffic. The gateway
does not become an agent — it fronts them — so it takes only
[`rusty_a2a`][rusty_a2a]'s protocol types.

### Method gating

An A2A call names its operation in the JSON-RPC `method` field, so a route can
permit `message/send` and refuse `tasks/cancel`. Denies win, and an empty allow
list means "everything not denied" — the same reading as the MCP tool gate.

A refusal is a **JSON-RPC error object with HTTP 200**, carrying the spec's
`PermissionDenied` code (`-32011`) and echoing the caller's request id. That is
where a JSON-RPC client looks; an HTTP error status would surface as a
transport failure rather than the reason.

A body that is *not* JSON-RPC passes through untouched. A2A also has REST and
gRPC bindings, and refusing everything this gate cannot read would break them
for no security benefit — the gate refuses named methods, it is not a schema
validator.

### Agent card discovery

With `agentCard` set, the gateway serves a merged card at
`/.well-known/agent-card.json`. **The URL rewrite is the point**: an agent
behind a gateway advertises its own address, and a client that reads the card
verbatim goes straight around the gateway, past its authentication, rate limits
and audit trail.

Skills across agents are unioned. Capabilities are **intersected** rather than
unioned — advertising `streaming` because one agent supports it sends a
streaming client to an agent that cannot, and the failure lands on the client.
A skill id offered by two agents is listed once and reported at startup rather
than renamed: unlike an MCP tool name, a skill id is descriptive rather than
what a caller invokes, so qualifying it would misrepresent the agent.

Cards are parsed **leniently**. `rusty_a2a`'s types are transliterated
field-for-field from the normative proto with required fields enforced —
correct for an agent, awkward for a gateway aggregating cards from agents it
does not control. One non-conformant agent is excluded and reported rather than
breaking discovery for the rest. If *no* card can be assembled the well-known
path answers `503`, because serving a half-built card would be worse than
admitting there is none.

Without `agentCard`, the gateway has no opinion about discovery and the
well-known path is proxied like any other request.

**What a merged card does not promise:** the union of skills describes what is
reachable behind the route, not a routing table. This gateway load-balances
across backends by weight and does not route by skill, so a client picking a
skill is not guaranteed to reach the agent offering it. With one backend — a
gateway fronting a single agent — the question does not arise, and the card is
that agent's with its URL corrected.

[Agent2Agent]: https://a2a-protocol.org

## The LLM gateway

```yaml
- name: llm
  matches:
    - path:
        pathPrefix: /v1
  policies:
    backendAuth:
      key: sk-ant-...          # the provider credential, not the caller's
  backends:
    - ai:
        provider:
          anthropic:
            model: claude-sonnet-4   # overrides whatever the caller asked for
```

A client POSTs an ordinary OpenAI chat-completions request and gets an OpenAI
response back, whichever provider actually served it. Switching provider is a
configuration edit rather than a client change — which is the entire point.

### What gets translated

For an OpenAI-compatible provider the body is forwarded essentially unchanged:
only `model` is overridden and the credential swapped. That is deliberate. A
typed round-trip would silently drop every field this gateway has not heard of
— tool definitions, `response_format`, `logprobs`, whatever ships next — and a
gateway that quietly deletes half a request is worse than one that refuses it.
`hostOverride` points the same path at any OpenAI-compatible endpoint,
self-hosted or otherwise.

Anthropic gets a real translation, because three differences bite:

- **`max_tokens` is optional for OpenAI and required by Anthropic**, so a valid
  request would be rejected. Translation supplies a default (4096) rather than
  passing the absence through.
- **The system prompt is a message for OpenAI and a top-level field for
  Anthropic.** Left in the message list it is either rejected or, worse,
  silently treated as a user turn. Several system messages are joined rather
  than dropped.
- **Finish reasons use different vocabularies** — `end_turn` is what OpenAI
  calls `stop`. A client switching providers should not have to learn both.
- **Tool calling is spelled differently at every step**, which is its own
  section below.

Gemini gets one too, and differs in ways the other two do not: see
[Gemini names the model in the URL](#gemini-names-the-model-in-the-url).

### Gemini names the model in the URL

```yaml
backends:
  - ai:
      provider:
        gemini: {}
```

OpenAI and Anthropic both name the model in the request body, so the address a
route dials is fixed and is resolved once at startup. Gemini puts it in the
path — `/v1beta/models/{model}:generateContent` — which makes the URL a
function of what the caller asked for, built after the route's `modelAliases`,
`defaults` and the backend's own `model` have all had their say.

That means a **client-controlled string ends up in a URL this gateway signs
with its own key**, so it is checked rather than trusted. A name may hold
letters, digits, dots, dashes and underscores, with an optional `models/`
prefix, and anything else is a 400 that never leaves the gateway. A `model` of
`../../v1beta/tunedModels/private` would otherwise point an authenticated call
at an endpoint nobody configured.

The credential goes in `x-goog-api-key` rather than the `?key=` query parameter
the API also accepts. A credential in a URL is written to every access log
between here and Google, and to this gateway's own if a request is ever traced.

The rest is rearrangement, in the same spirit as Anthropic's:

- **A turn is `parts`**, even when it is one string, and the roles are `user`
  and `model` rather than `user` and `assistant`. Two turns of one role in a
  row are joined, because Gemini rejects them and a caller that sent two user
  messages meant both.
- **The system prompt is `systemInstruction`**, with no role of its own.
- **Sampling settings live in `generationConfig`** — `maxOutputTokens`, `topP`,
  `stopSequences`. Carried over by name they would produce a request Gemini
  accepts and quietly ignores, which is worse than one it rejects.
- **Refusal finish reasons become `content_filter`.** `SAFETY`, `BLOCKLIST`,
  `PROHIBITED_CONTENT` and `RECITATION` have no OpenAI equivalent, and
  reporting them as a clean `stop` would tell a client the model finished its
  thought when it was cut off.
- **An image part is dropped.** Gemini takes bytes as `inline_data` or a URI as
  `file_data` and never a URL to fetch, so an OpenAI `image_url` has no honest
  translation. Text parts carry across.

#### A Gemini stream is a different method, and its frames are whole

`stream: true` is not a field there. It is `:streamGenerateContent?alt=sse` — a
different URL, and without `alt=sse` the API answers with a JSON *array* of
responses rather than server-sent events.

The frames themselves are not deltas. Anthropic sends named events carrying
fragments of a message; Gemini sends a whole `GenerateContentResponse` each
time, with no event name, holding what was produced since the last one. So it
gets a translator of its own, and the only state that translator keeps is what
OpenAI repeats on every chunk — the id, the model, and whether the assistant
role has been announced.

**A streamed tool call arrives in one piece.** Anthropic and OpenAI both send a
call's arguments as fragments the client concatenates; Gemini sends the
`functionCall` complete. So the chunks here carry the entire `arguments` string
at once. That is valid — concatenating one fragment gives the same result — but
it is a real difference in what goes over the wire, worth knowing before
reading a packet capture and wondering where the deltas went.

`usageMetadata` is cumulative rather than incremental, so the latest frame to
carry one replaces what came before. Summing them would bill a long answer
several times over.

#### Tool calling on Gemini

The same four legs [Tool calling](#tool-calling) below describes for Anthropic,
with two problems that side does not have.

**A result is matched by name, not by id.** OpenAI's tool result carries a
`tool_call_id` and no name; Anthropic's `tool_result` carries the same id, so
that leg is a rename. Gemini's `functionResponse` carries the **function's
name**, and the call it answers carries no id at all. So the messages are
walked in order, remembering each call's id and name as they go. A result whose
id was never announced is dropped rather than sent under a guessed name — a
`functionResponse` naming the wrong function is answered as though the wrong
tool ran.

The ids in that conversation are ones this gateway made up, positionally
(`call_0`, `call_1`), because Gemini sends none and an OpenAI client needs one
to answer under. A client that echoes back what it was given round-trips
exactly.

**A parameter schema has to be cut down.** Gemini's `Schema` is a subset of
JSON Schema and its parser *rejects* fields it does not know rather than
ignoring them. An OpenAI tool definition written for strict mode carries
`additionalProperties: false`, which on its own fails the whole request with
`Unknown name "additionalProperties"`. So the schema is filtered to the fields
Gemini defines, recursively — a nested object's rejected field fails the request
exactly as an outer one does — and `$schema`, `$ref`, `oneOf`, `allOf` and
`const` go the same way.

The filter is an allow-list rather than a deny-list because the two failures are
not symmetric: a dropped constraint loosens validation the model was going to
treat as advice anyway, while one kept unknown field refuses the call outright.

Everything else is a rename: `functionDeclarations` inside a `tools` wrapper —
Gemini's `tools` is a list of *kinds* of tool, not of functions — and
`tool_choice` becomes `toolConfig.functionCallingConfig`, where naming a single
function is `ANY` narrowed to it with `allowedFunctionNames`.

### Tool calling

A tool call is a round trip, and Anthropic spells every leg of it differently:

| | OpenAI | Anthropic |
| --- | --- | --- |
| definition | `{type: function, function: {…, parameters}}` | flat, with `input_schema` |
| choice | `auto` / `none` / `required` / a named function | `{type: auto}` / `{type: none}` / `{type: any}` / `{type: tool, name}` |
| the call | `tool_calls` beside the assistant's text | a `tool_use` **content block** |
| the result | a message with `role: "tool"` | a `tool_result` block inside a *user* turn |

All four are translated. Two details are easy to get wrong and are worth
stating:

**Consecutive tool results join one user turn.** Anthropic rejects two user
messages in a row, and a model asked to call three tools gets all three answers
before the conversation moves on.

**`arguments` is a string on one side and an object on the other.** Translating
either way means parsing or serializing something a *model* produced, so a
partial or malformed argument string becomes an empty object rather than an
error — failing a whole conversation over one garbled call is worse than
forwarding a call the model can be told went wrong.

Streamed calls need a second index. Anthropic numbers *content blocks*, and
text shares that numbering; OpenAI numbers *tool calls*, and its text is not in
the list at all. A response whose first block is text and whose second is a
call has that call at Anthropic index 1 and OpenAI index 0 — passing the block
index through would leave a client assembling arguments into a call that never
opened.

Argument fragments are forwarded as they arrive rather than assembled here.
They are not valid JSON on their own, and holding them back until the call
closed would defeat the point of streaming it.

### Streaming

Streams are re-framed, not buffered: an LLM response is the one thing a client
most wants incrementally, and collecting it would turn a token-by-token answer
into a long silence followed by a wall of text.

OpenAI frames pass through byte-identically. Anthropic's event stream is
translated by a small state machine, because the two are not a per-event
mapping: Anthropic sends the id, model and *prompt* tokens once in
`message_start` and the *completion* tokens in `message_delta`, while OpenAI
repeats id and model on every chunk, announces the assistant role exactly once,
and terminates with a literal `data: [DONE]` that is not a chunk at all.

Token usage is reported as a structured log line rather than a metric —
per-request counts keyed by model are unbounded label cardinality, and a client
inventing model names should not be able to take a metrics backend down.

### Errors are passed through

A provider's status and body are returned as they arrived. "invalid api key" is
the useful part, and a gateway that rewrites it as "bad gateway" costs an
afternoon. Errors the gateway itself generates use OpenAI's error envelope, so
a client's existing handling works.

`backendAuth: passthrough` is ignored here: a provider API key is not the
caller's bearer token, and forwarding one as the other would send a user's
credential to OpenAI.

### Shaping the request: the `ai` policy

```yaml
policies:
  ai:
    modelAliases:
      fast: gpt-4o-mini          # a name callers may use
    prompts:
      prepend:
        - role: system
          content: House rules.  # on every call, whatever the client sent
    defaults:
      temperature: 0.2           # only when the caller left it out
    overrides:
      max_tokens: 512            # whatever the caller asked for
```

Four of these can touch the same field, so **which wins is a stated ladder**
rather than something to discover:

1. `modelAliases` resolves the name the caller used. First, because it is about
   what the caller *meant* — everything below should see the resolved name.
2. `prompts` shape the conversation.
3. `defaults` fill in what the caller left out, and only that.
4. `overrides` replace what the caller or the defaults set.
5. The backend's own `model:` still wins over all of it. It is backend
   configuration rather than route policy — the most specific statement about
   where traffic goes — and it was already the rule before any of this existed.

Read downwards, each step is "more specific wins", which is the only ordering
an operator can predict without reading the code.

All of it runs on the **OpenAI-shaped body, before translation**. That is the
only place a rule written once means the same thing for every provider: after
translation there is no `messages` array to prepend to, because Anthropic has
hoisted the system prompt out of it.

Two smaller decisions. An alias is resolved **once**, so `a → b → c` gives `b`;
a chain would let a config loop, and a gateway that hangs on its own
configuration is worse than one that stops after a step. And a body with no
`messages` is **left alone** rather than given some — inventing the array would
turn a client bug into a request that runs with only the operator's prompt in
it.

### Prompt caching

```yaml
policies:
  ai:
    promptCaching:
      cacheSystem: true
      cacheMessages: true
      cacheMessageOffset: 1     # behind the turn that changes
      minTokens: 2048
```

The one part of the policy that runs *after* translation, because a cache
breakpoint is a provider-specific annotation on a provider-specific shape —
Anthropic's `cache_control` on a content block. A string `system` prompt is
promoted to a block so it can carry one; a list is marked on its last block,
since the breakpoint covers everything up to where it sits.

**Only Anthropic.** OpenAI caches long prefixes by itself and takes no
configuration for it, so this is a no-op there rather than an error — a route
that sets it and later switches provider should not stop starting.

`minTokens` is an optimisation, not a correctness rule: a provider will not
cache a short prefix anyway and ignores the marker. That is why the length is
**estimated** at roughly four characters per token rather than tokenised — a
real tokeniser would mean shipping a vocabulary per model to decide whether to
add an annotation that is free to get wrong.

`cacheTools` marks the last tool definition. Tools sit ahead of the system
prompt and the conversation in what Anthropic caches, so a breakpoint there
covers the least that changes between calls — the cheapest one to set and the
likeliest to hit.

### Prompt guards

Content rules over what a caller sends and what comes back. Three kinds, in
increasing order of what they can notice and what they cost: a `regex` pattern
written down in advance, a `webhook` asking a service the operator runs, and
`openAIModeration` asking a classifier.

```yaml
policies:
  ai:
    promptGuard:
      request:
        - regex:
            action: reject
            rules:
              - pattern: "password[=:]\\s*\\S+"
          rejection:
            status: 422
            body: '{"error": {"message": "no credentials, please"}}'
      response:
        - regex:
            action: mask
            rules:
              - builtin: phoneNumber
```

A rule either **rejects** — the request stops and the operator's own body goes
back — or **masks**, replacing what matched and carrying on. Rules run in order
and the first refusal ends it, so a list can be read top to bottom. A refusal
with no `status` is a **400**: a content rule decides the request is
unacceptable *for this route*, which is what a bad request means; `403` would
send someone to check credentials that are fine.

Builtins are `email`, `phoneNumber`, `ssn`, `creditCard` and `caSin`, and each
says what it found — `<PHONE_NUMBER>`, `<SSN>`. A pattern you write yourself
becomes `<masked>`, because there is nothing more specific to say about it. A
pattern that does not compile is a **startup failure**, since the alternative
is a rule that silently never fires — which reads exactly like content nobody
sent.

Request rules scan what the *caller* sent, before `prompts` add anything: an
operator refusing their own system prompt would be a strange thing to arrange.
Every message is scanned, not just the last, because a credential three turns
back is still on its way to the provider. A tool call's arguments are
deliberately **not** scanned — they are a structured object built from a schema
the operator wrote, and masking inside one produces JSON that no longer matches
the tool it will be called with.

#### Asking a service instead of a pattern

```yaml
policies:
  ai:
    promptGuard:
      request:
        - webhook:
            target:
              host: guard.internal:9000
            headers:
              ":path": '"/api/guardrails/request"'
              x-tenant: 'request.headers["x-tenant"]'
              x-user: jwt.sub
            forwardHeaderMatches: [x-trace]
            failureMode: failClosed
```

Where `regex` decides from a pattern written down in advance, a `webhook` asks
something that can change its mind — a classifier, a policy service, a model.

**The wire contract is upstream's, read from upstream's source.** It is not in
agentgateway's published documentation: the guardrail pages describe how to
*configure* a webhook and link to an API reference that does not render. The
shapes come from `crates/agentgateway/src/llm/policy/webhook.rs` in the
upstream repository, so an existing webhook works here unchanged — guessing
would have quietly broken the one thing this project is for.

The gateway `POST`s to `/request` and `/response`, with the conversation or the
answer:

```json
{"body": {"messages": [{"role": "user", "content": "..."}]}}
{"body": {"choices": [{"message": {"role": "assistant", "content": "..."}}]}}
```

and reads back one `action`, told apart by **shape** rather than a tag, because
upstream's enum is untagged:

```json
{"action": {"reason": "..."}}                               // pass
{"action": {"body": {"messages": [...]}, "reason": "..."}}  // mask
{"action": {"body": "text", "status_code": 403}}            // reject
```

A `mask` carries an *object* body and a `reject` a *string* one, so the two
cannot be confused; `pass` is what is left. A mask this build cannot read is a
**refusal**, not a pass — the webhook asked for a rewrite and did not say to
what, and serving the original would serve exactly the text it objected to.

**It fails closed.** Unreachable, timed out, a non-2xx answer, or a body with
no `action` all refuse unless `failureMode: failOpen` says otherwise; a content
control that waves traffic through when its service is down is not one. The
refusal is a **503**, not a 400: nothing decided the content was unacceptable,
and saying so would send someone to inspect a prompt that is fine.

Header expressions are CEL over the **caller's** request — `request.headers`,
`jwt.*` (the claims `jwtAuth` verified, which a caller cannot forge), and
`llmRequest.*`. Setting `:path` moves the call. An expression that resolves to
nothing sends **no header** rather than an empty one, since an empty `x-tenant`
claims there is no tenant. `forwardHeaderMatches` is an allow-list and empty
forwards nothing — the opposite of `mcpGuardrails`, and deliberate: the body
already carries the prompt, so headers are extra reach rather than the point.

Rules run in order, so putting a cheap `regex` rule above a webhook saves the
network call when it refuses.

#### Asking OpenAI's classifier

```yaml
policies:
  ai:
    promptGuard:
      request:
        - openAIModeration:
            model: omni-moderation-latest    # the default
          rejection:
            status: 451
```

A model trained to recognise categories no pattern describes — harassment,
self-harm, sexual content, violence. The endpoint answers a verdict per input
with a `flagged` flag on each, and **any** flag refuses, without looking at
which category: picking categories apart would be inventing a policy language
upstream does not have.

It classifies the **prompt only**. Upstream's response guard has no moderation
variant, and a rule written under `response` is reported by `--check` rather
than quietly inspecting nothing:

```
warning: ...promptGuard.response[0].openAIModeration: moderation classifies a
         prompt, and upstream accepts it on the request phase only, so this
         rule inspects nothing
```

The text it is shown is every message's content, in order — including the text
parts of a multimodal message, which is more than a `regex` rule reads. The two
differ because this one only *reads*: masking rewrites what it matched, and
rewriting inside a content part means rebuilding a structure whose other parts
are images. Reading carries no such risk, and skipping the list form would let
a prompt evade the rule by spelling itself the other way. A prompt with no text
at all is allowed without a call, since an empty input list only earns a 400.

It **fails closed**. A classifier that cannot be reached, times out, or answers
unreadably refuses with a **503** — not the rule's own rejection, because
nothing decided this prompt was unacceptable and a content refusal would send
someone to inspect a prompt that is fine. Upstream's moderation carries no
`failureMode`, unlike a webhook, so there is nothing to configure and no
fail-open path.

##### Which key it calls with, and where that key travels

A moderation call is an authenticated call to OpenAI, so it needs an OpenAI
key. Two ways to have one:

```yaml
- openAIModeration:
    policies:
      backendAuth:
        key: sk-moderation      # its own, and it calls api.openai.com
```

or nothing at all, in which case the rule borrows the **route's**
`backendAuth.key` — but only when the route's provider is `openAI`. A route on
Anthropic has a key too, and borrowing it would send an Anthropic credential to
OpenAI: a secret handed to a third party by a gateway the operator trusted to
do the opposite. So a moderation rule on a non-OpenAI route without its own key
**does not start**:

```
route[0].ai.promptGuard.request[0].openAIModeration: moderation calls OpenAI
and has no key to call it with; give the rule a `policies.backendAuth.key`,
because an `anthropic` route's own key is not an OpenAI credential and sending
it to OpenAI would hand a secret to a third party
```

`passthrough` is refused for the same family of reasons the `ai` backend
refuses it: a client's bearer token is not an OpenAI API key.

A borrowed key travels **no further than the route it came from** — the call
goes to that route's own base URL, `hostOverride` included, rather than to
`api.openai.com`. This is a deliberate, narrow divergence from upstream, which
always calls `api.openai.com`, and it exists for the reason above: a key issued
for one host should not be sent to another. An operator who wants the real
OpenAI from a route pointed elsewhere says so by giving the rule its own key.

#### A response rule buffers a streamed answer

This costs something a caller will notice, so it is worth being plain about.

A pattern can straddle a chunk boundary: `"call 555-"` arrives, then
`"867-5309"`. Scanning each chunk alone misses that, and by the time the second
shows what the first started, the first is already at the client and cannot be
recalled.

A sliding window — hold back the last N bytes, scan across the join — keeps the
stream, but has to pick N, and a regex has no general longest-match bound:
`\d+` runs past any window. The failure would be a **silent leak of exactly
the thing the rule exists to catch**, which is worse than an obvious cost.

So a route with a response rule collects the whole answer, checks it, and then
sends it — as one content chunk, since after masking it is no longer the text
the provider chunked and inventing boundaries for it would be making something
up. A **request** rule costs a stream nothing, and only `response` triggers
this. Startup logs it on the route that has it.

A refusal on the response side is an ordinary JSON error rather than an event
stream: nothing has gone out yet, so the client can still be told plainly.
Guarding also gives up the byte-for-byte OpenAI passthrough, since rewriting
means re-serializing — the cost of asking for the answer to be inspected.

### What the `ai` policy does not do yet

`--check` names the sub-policies one at a time rather than reporting `ai` as a
whole:

```
warning: ...policies.ai.routes: parsed but not enforced by this build
```

One finding for the whole policy was accurate while none of it was implemented
and would be a lie now — an operator who reads "not enforced", then watches
their `prompts` work, has no idea what else is being ignored. That granularity
goes all the way down: a rule kind this build does not run, beside a `regex`
one in the same list, reports only itself.

Every named sub-policy is implemented now. What is left is whatever upstream
adds next, which lands in the catch-all and is reported by name — and the one
placement the schema allows but upstream does not, `openAIModeration` under
`response`.

## TLS

```yaml
binds:
  - port: 8443
    listeners:
      - protocol: HTTPS
        tls:
          cert: /etc/certs/tls.crt   # chain, leaf first
          key: /etc/certs/tls.key    # PKCS#8, PKCS#1 or SEC1
        routes: [...]
```

Termination goes through [`rusty_tls`][rusty_tls], the ecosystem's one TLS
implementation, so the gateway does not roll its own. ALPN advertises `h2` and
`http/1.1`; over TLS that is how the HTTP version gets chosen, and a client
capable of HTTP/2 gets it.

Certificates are read at startup, so a missing or malformed one stops the
gateway booting instead of failing every handshake later. `X-Forwarded-Proto`
reports `https` for a TLS listener — an upstream generating absolute URLs from
that header would otherwise emit `http://` links into an `https://` page and
trip mixed-content blocking.

### A certificate per hostname

Two listeners on one port may hold different certificates, chosen by the name
the client asked for:

```yaml
binds:
  - port: 8443
    listeners:
      - hostname: api.example.com
        protocol: HTTPS
        tls: {cert: /etc/certs/api.crt, key: /etc/certs/api.key}
        routes: [...]
      - hostname: "*.internal.example.com"
        protocol: HTTPS
        tls: {cert: /etc/certs/internal.crt, key: /etc/certs/internal.key}
        routes: [...]
```

**The name is read off the ClientHello rather than resolved by `rustls`.**
`rusty_tls` builds its acceptor from a single chain and holds its `ServerConfig`
privately, so there is no `ResolvesServerCert` to install — and reaching around
the crate into `rustls` to build a config by hand would give up the one thing
importing it buys, that this gateway is not the consumer in the ecosystem
rolling its own TLS.

So the first bytes a client sends are **peeked**, not read: the SNI extension is
plaintext in the ClientHello, because a server cannot decrypt anything before
it knows which certificate to present. The name chooses the acceptor, and the
bytes are still in the socket for `rustls` to parse a moment later.

Nothing about that decides whether a handshake succeeds — `rustls` still does
all of it, on the same bytes. A ClientHello this cannot parse returns no name,
the default certificate is served, and the handshake works or fails on its own
merits. That is exactly the behaviour from before selection existed, which is
what makes a permissive parser the safe choice: every length in the message is
written by whoever connected, so every read is bounds-checked against the slice
rather than against the length that preceded it.

Exact names beat wildcards beat the catch-all, the same precedence route
hostnames follow. A client sending no name at all — every one addressing the
gateway by IP — and one asking for a name nothing claims both get the first
certificate rather than a refusal; refusing would turn a working
single-certificate deployment into a broken one the moment a second listener was
added.

Two things are still startup errors, because a name cannot choose between them:
two listeners claiming the *same* hostname with different certificates, and two
certificates on a port where neither listener names a hostname. The same
certificate on two listeners is one certificate and is fine.

### Forwarding without terminating

A listener carrying `tcpRoutes` passes connections through instead. Nothing is
decrypted, no certificate is presented, and the gateway never learns what is
inside:

```yaml
binds:
  - port: 8443
    listeners:
      - protocol: TLS
        tcpRoutes:
          - hostnames: ["alpha.example.com"]
            backends:
              - host: "10.0.0.1:443"
          - hostnames: ["*.internal.example.com"]
            backends:
              - host: "10.0.0.2:443"
          - backends:                      # the catch-all
              - host: "10.0.0.9:443"
```

`tcpRoutes` is what says passthrough, not the protocol: `TLS` covers both of
the Gateway API's modes, and a listener naming it with a certificate and HTTP
routes has been terminating since before passthrough was possible here. A
listener with both is reported by `--check` — a port either forwards
connections or serves requests on them.

**The only thing such a route can match on is the name.** A path, a method, a
header: all of them are inside the encryption. So a TCP route is hostnames and
backends and nothing else, matched from the same peeked ClientHello that
selects a certificate on a terminating listener. Exact names beat wildcards
beat the catch-all, and backends are weighted and load-balanced by the same
ring everything else uses.

A connection carrying no name — a client dialling by IP, or something that is
not TLS at all — matches only a route with no hostnames. A name nothing claims
is **closed**, because forwarding it to whichever route sorted first would send
a connection somewhere the operator never pointed that name.

**This is a different trust boundary, and `--check` says so:**

```
warning: ...listeners[0]: forwards connections without decrypting them, so no
         route policy applies to this traffic -- not `jwtAuth`, `extAuthz`,
         header modifiers or guards; authorization here is whichever names the
         `tcpRoutes` claim
```

Every policy in this gateway works because the bytes are in the clear at some
point, and none of them are here. That is invisible in a file that simply has
no `policies:` key, which is why it is said out loud rather than left implied.

## Retries and rate limits

```yaml
policies:
  retry:
    attempts: 2          # retries *after* the first try, so three tries total
    backoff: 100ms       # doubles each attempt, capped at 30s
    codes: [502, 503]
  localRateLimit:
    - maxTokens: 100     # burst
      tokensPerFill: 100
      fillInterval: 60s
    - maxTokens: 5       # sustained
      tokensPerFill: 5
      fillInterval: 1s
```

### What is safe to retry

A **connect** failure never reached the upstream, so replaying it cannot
duplicate work. Any other transport error is ambiguous — the request may have
arrived and been processed, with only the response lost on the way back — so it
is **not** retried. Replaying that would silently double a payment or a write.
A timeout is ambiguous for the same reason and is not retried either.

A **status code** is different: the upstream answered, so it certainly saw the
request. Retrying is the operator's explicit choice by listing the code, which
is why nothing is retried on status unless `codes` names it. Each attempt takes
the next endpoint in the ring, since retrying the instance that just failed is
the least likely way to succeed.

### Why a body has to be buffered

A retry replays the request, and a streaming body can only be read once. A body
is buffered only when its size is **known in advance** and fits in 64 KiB — so
the body is never partially consumed to find out how big it is, which would
leave a stream that can be neither replayed nor forwarded. Requests with a
`Content-Length` inside the limit get retries; chunked or oversized ones are
streamed straight through and simply do not.

### Which backends retry

`retry` used to be consumed only by the `host` proxy, so an `ai` route asking
for three attempts got exactly one.

An **`ai`** route retries now, on the same policy and the same rules — a listed
status, or a connect failure, and nothing else. Buffering is not a question
there: an `ai` request has to be read to be translated, so it is replayable by
construction and `attempts` always means what it says. The body is serialized
once and replayed, so a retry costs a request rather than a re-encode.

```yaml
policies:
  retry:
    attempts: 2
    backoff: 500ms
    codes: [429, 503]     # the two a model provider actually sheds with
  backendAuth:
    key: sk-...
backends:
  - ai:
      provider:
        openAI: {}
```

Streaming is unaffected. The retry decision is made on the response head, which
arrives before the first token, so a stream that gets retried has not started
coming back yet. Exhausting the attempts returns **the provider's own last
answer**, not a gateway error — the message is the useful part, and rewriting
"rate limit exceeded" into "bad gateway" costs an afternoon.

An **`a2a`** route already retried, through the same proxy. It is also always
replayable, for a different reason: the policy has to buffer the body to read
the JSON-RPC method out of it, so by the time the proxy sees the request it is
buffered whatever its size. A refused method is never retried at all, because
it was never sent — the refusal is the gateway's own response.

An **`mcp`** route is reported rather than retried:

```
warning: ...policies.retry: an `mcp` backend holds a session rather than making
         a request it could make again, so `codes` names statuses nothing here
         returns, and replaying a `tools/call` after an ambiguous transport
         error would run the tool twice; it is not applied
```

Both halves of that matter. `codes` are HTTP statuses, and an MCP route sends a
JSON-RPC message over a session it already holds — there is no HTTP response at
that layer to read one from. And the safety rule everything else leans on, that
a *connect* failure is the one error known never to have reached the upstream,
has no equivalent: a transport error on an established session covers both
"never sent" and "sent, reply lost". Retrying under that ambiguity is how a tool
whose entire purpose is a side effect performs it twice.

### Rate limits

Several limits on one route must *all* permit a request, which is how burst and
sustained rates are expressed together. They share one lock: checking them in
turn would spend a token from the burst bucket before discovering the sustained
one refuses, billing a request that was never served.

Refill is lazy — no background task ticks every bucket in the process — and
credits whole elapsed intervals only, because advancing to *now* on each check
would discard the remainder every time and refill slower than configured.
Buckets start full, so a gateway that has just restarted does not spend its
first interval refusing traffic.

Over the limit is `429` with `Retry-After`, reporting the *longest* wait across
the limits that refused: coming back when only the shorter one has refilled
would just be refused again. `Retry-After` is never `0`, which would invite an
immediate retry.

Rate limiting runs **before** authentication, so a flood is refused before it
costs a signature verification and possibly a JWKS fetch. It runs after the
CORS preflight branch for the same reason authentication does — a browser
reports a refused preflight as an opaque CORS error, hiding the 429 the caller
needs to see.

### Limiting LLM tokens rather than requests

`type: tokens` counts what a model provider reported, not how many calls were
made. On an `ai` route it is usually the limit that matters: ten requests are
cheap and one 200k-token context is not.

```yaml
policies:
  localRateLimit:
    - maxTokens: 60            # 60 requests a minute
      tokensPerFill: 60
      fillInterval: 60s
    - maxTokens: 200000        # and 200k LLM tokens an hour
      tokensPerFill: 200000
      fillInterval: 1h
      type: tokens
backends:
  - ai:
      provider:
        openAI: {}
```

The two kinds coexist and are charged at different moments, which is the whole
reason they are separate limiters over the same buckets. A request limit is
charged **before** dispatch, where it applies to every backend kind. A token
limit cannot be: **the cost of a call is not knowable until the provider
reports it.** So a request is admitted while the bucket has anything left, and
the actual count is charged afterwards.

Two consequences worth stating plainly. **One call can exceed the limit** — the
budget could have 1 token left and a 50k-token call still goes through. That is
inherent to charging a cost nobody knew in advance; the bucket is what stops the
*next* one. And a call that costs more than the bucket held **empties it rather
than going into debt**, so the overshoot is paid for by the wait until the next
refill and not carried forward into it. The alternative — reserving an estimate
up front and reconciling — refuses real traffic on a guess.

Streamed responses are charged too. Usage arrives in the trailing chunk, long
after the response body has started going back to the client, so the limiter
travels with the stream. A limit that only applied to buffered responses would
miss most of the traffic worth limiting.

Anywhere other than an `ai` route, a `type: tokens` limit is **reported**:

```
warning: ...policies.localRateLimit[type=tokens]: only an `ai` backend reports a
         token count to charge, so this bucket would never be spent; use
         `type: requests` to limit this route
```

It would sit at full capacity and refuse nothing — a rate limit that looks like
protection and is not.

## Timeouts

The two budgets in a `timeout` policy bound genuinely different things, and the
difference is not cosmetic:

```yaml
policies:
  timeout:
    requestTimeout: 30s          # time to *produce* a response
    backendRequestTimeout: 10s   # time for an upstream call to finish
```

Measured against a tool that sleeps five seconds, `time_starttransfer` is ~1ms
and `time_total` is ~5s: the Streamable HTTP transport sends its SSE response
headers immediately and streams the JSON-RPC result afterwards. So by the time
a tool starts running, `requestTimeout` has already been satisfied — **it
cannot cut a tool call off, and a route that sets only `requestTimeout` has no
bound on how long a tool may run.**

That is deliberate rather than a defect: bounding the whole stream would sever
every long-lived subscription. `backendRequestTimeout` is the budget that
bounds a tool call, applied around the upstream request inside the federation.
A call that exceeds it comes back as a tool error naming the tool and the
budget — a result the model can read, not an opaque protocol failure. It also
bounds `tools/list`, so one hung target cannot hold up the whole catalogue.

There are tests pinning both halves, including one asserting that
`requestTimeout` does *not* kill a long stream, so nobody "fixes" it into
something that severs live subscriptions.

## Observability

```yaml
config:
  tracing:
    endpoint: http://localhost:4317
    serviceName: rusty-agent-gateway
    sampleRatio: 1.0
  limits:
    maxConcurrentRequests: 256
    requestTimeout: 30s
```

With `tracing` set, spans and metrics go to an OTLP collector. Sampling is
parent-based: a caller that already sampled the trace is followed, because
deciding independently is how traces end up half-recorded with gaps exactly
where a service made its own choice.

MCP request metrics are labelled from the `Mcp-Method` and `Mcp-Name` headers,
so no request body is parsed. The tool-name label is bounded by the names each
route actually federates — anything else is labelled `other`. Without that
bound, a client could mint unbounded time series by calling names that do not
exist, which is how a metrics backend gets taken down from the outside.

`maxConcurrentRequests` sheds with `503` and `Retry-After` rather than queueing:
a queue in front of an overloaded gateway turns a capacity problem into a
latency problem, where every client waits longer, times out and retries — which
is how a brief spike becomes a sustained outage. It counts requests until a
response is *produced*, so as above it bounds handshakes, authentication and
JWKS fetches rather than streaming tool calls. That is the load worth shedding:
an unauthenticated flood costs a token validation each.

Both limits are off unless configured. There is no concurrency number right for
everyone, and a default would be a silent regression for a gateway already
serving more than it.

## On `rusty_mcp`

[`baileyrd/rusty_mcp`][rusty_mcp] was evaluated for reuse here. It is a
server-side scaffold on `rmcp` 3.x, and the overlap is real but partial.

**Not reused: its `runtime` and `cli` layer.** `rusty_mcp::serve` binds a socket
and owns one server on one transport at one path. A gateway is
`binds[] → listeners[] → routes[]`, with MCP as one backend kind among several,
so it needs a mountable `tower` service rather than something that binds for
you — which is `rmcp`'s `StreamableHttpService` directly, and what this repo
uses.

The deeper mismatch is protocol version. `rusty_mcp` pins **2026-07-28** and
defaults to `NeverSessionManager`, deliberately: that revision is stateless, and
the payoff is horizontal scaling with no session affinity. A gateway cannot take
that position. It has to speak whatever its clients speak and whatever each
target speaks, and today's ecosystem is largely 2025-06-18/2025-11-25 with
sessions — which is why upstream's own quickstart puts `Mcp-Session-Id` in
`exposeHeaders`, and why this gateway uses `LocalSessionManager`.

**Reused: its `auth` module**, as a git dependency pinned to `v0.4.1` (it is
`publish = false`, so there is no crates.io path). `JwtValidator` brings JWKS
caching with a TTL, an algorithm allow-list pinned *before* any key is loaded —
which is what defeats the `alg: none` and RS256-verified-as-HS256 attacks — and
a floor between refetches provoked by an unknown `kid`, without which anyone can
force unbounded outbound requests to the authorization server by presenting
tokens with random `kid` values.

Two things it does not cover, supplied by `agentgateway-auth`: the audience
check described above, and a file-backed JWKS, since `JwtValidator` fetches over
HTTP only. `FileJwks` mirrors its verification path deliberately, including the
ordering that matters. If `rusty_mcp` grows a `JwtValidator::from_jwks`
constructor, that module should collapse into it — duplicated crypto is
duplicated risk, and this is the one place this repo carries any.

**Also reused: `otel` and `limits`.** `otel::init` stands up the OTLP pipeline
and installs the subscriber; `OtelGuard` is held until after the accept loops
stop, because spans are batched and whatever is still buffered dies with the
process otherwise. `McpMetricsLayer` supplies the MCP request metrics described
above. `LimitsLayer` supplies the concurrency bound, mounted outside routing and
authentication so a shed request costs a semaphore try-acquire rather than a
route lookup, a token validation and a JWKS fetch.

One correction worth recording: `limits` is concurrency and timeout shedding,
**not** a token bucket, so it does not cover `localRateLimit`
(`maxTokens`/`tokensPerFill`/`fillInterval`). That policy needed its own bucket,
and has one — `agentgateway-core`'s `RateLimiter`, described under
[Retries and rate limits](#retries-and-rate-limits).

Its lint posture (`missing_docs = warn`, `unsafe_code = forbid`,
`unwrap_used = warn`) is adopted verbatim in this workspace.

## On `rusty_tls`

TLS termination is [`rusty_tls`][rusty_tls] — a `rustls` 0.23 wrapper — rather
than `tokio-rustls`, so the gateway is not the one consumer in this ecosystem
that rolls its own TLS.

Its async server adapter is written against `rusty_tokio`'s `AsyncRead` and
`AsyncWrite`, while this gateway runs on `tokio` and hands connections to
`hyper`, so a TLS stream is adapted twice: the socket goes in as a
`rusty_tokio` stream and the decrypted stream comes back out as a `tokio` one.
**That costs nothing per byte.** Both runtimes' `ReadBuf` is an initialized
`&mut [u8]` plus a filled cursor, so the inner reader writes directly into the
outer buffer's spare capacity and the adapter only forwards the count — no
intermediate buffer, no copy, no `unsafe`. The price of the choice is
dependency weight (`rusty_tokio` for its trait definitions, and `rustils`'
`platform` crates that only client-side trust anchors use), not throughput.

One deliberate exception to the crate's "consumers import `rusty_tls`, never
`rustls`" rule: `rusty_tls` does not install a `rustls` `CryptoProvider`, and
`rustls` refuses to guess when more than one is present in a build. This
workspace has two — `ring` via `rusty_tls`, `aws-lc-rs` via `reqwest` — so it
panics on the first handshake. `agentgateway-tls` installs `ring` explicitly,
once, which needs a direct `rustls` dependency pinned to the same 0.23.

## On `rusty_a2a`

A2A support is [`rusty_a2a`][rusty_a2a] — a complete implementation of the
protocol: all 11 operations across JSON-RPC, gRPC and HTTP+JSON/REST, agent
card discovery, JWS signing and push notifications.

The gateway uses about a tenth of it, and that is the right amount. The crate's
centre of gravity is *being* an agent — `AgentExecutor`, task store, lifecycle,
streaming — while a gateway fronts agents. So this takes the protocol types and
leaves the harness alone.

That also settles what looked like the main integration cost. `rusty_a2a` pins
`axum` 0.7, `tonic` and `reqwest` 0.12 against this workspace's 0.8 and 0.13,
which would mean two copies of `reqwest` with separate TLS stacks. All of it is
behind the `client`, `server`, `grpc` and `signing` features, so
`default-features = false` pulls **45 crates with no `axum`, no `tonic` and a
single `reqwest` 0.13**. `build.rs` is gated on `grpc`, so no `protoc` either.

The one real friction is strictness, described under
[Agent-to-agent](#agent-to-agent): the types enforce required fields exactly as
the proto specifies, which is right for an agent and wrong for a gateway
aggregating third-party cards. Hence the lenient parse here.

## Building and testing

The toolchain is pinned in `rust-toolchain.toml`, so rustup installs the right
version on first use and a local check and CI check the same thing. That pin
exists because they once did not: clippy 1.97 rejected code 1.94 accepted, so a
clean local run passed while CI failed on the same commit twice running.

The three commands CI runs, which are the three worth running before a push:

```bash
cargo fmt --all -- --check
cargo clippy --all-targets --all-features -- -D warnings
cargo test --all-features
```

832 tests. The config tests are anchored on YAML taken verbatim from
agentgateway's documentation — if upstream examples stop parsing, compatibility
has regressed. The federation tests are genuinely end-to-end: a real `rmcp`
client, over a real socket, through the gateway, into real subprocess MCP
servers (`crates/agentgateway/examples/mock_mcp_server.rs`). Each fixture echoes
its own label in every result, so the tests assert a call reached the *right*
target rather than merely reaching one.

The JWT tests generate an RSA keypair at test time and sign real tokens, so
signatures are genuinely verified rather than stubbed — including a forged token
signed by an untrusted key and an `alg: none` token.

The timeout tests drive a genuinely slow subprocess rather than a stubbed
future, which is how the `requestTimeout`/`backendRequestTimeout` distinction
above was found rather than assumed.

The A2A tests run mock agents that serve real agent cards and record the calls
they received, so "was the method refused" is told apart from "did the agent
see it anyway". The LLM tests stand up a mock provider that records the body it
actually received, so the assertions are about whether the provider got the shape its
API requires rather than merely whether the gateway answered.

The proxy tests stand up a real upstream that echoes back the request line and
headers it saw, so the assertions are about what actually arrived rather than
what the gateway believed it sent. The retry tests script that upstream — "fail
twice, then succeed" — and count hits, which is how "did not retry" is told
apart from "retried and got the same answer".

The rate limiter takes time as a parameter rather than reading a clock, so its
tests drive time instead of sleeping. A rate limiter tested with `sleep` is one
tested at a single resolution, on one machine, when CI was not busy.

The TLS tests generate a certificate per name and give each client only its own,
so a handshake that succeeds for the wrong reason still fails the test. The
ClientHello parser is tested by truncating a valid message at every byte offset
and asserting that none of them parse or panic — every length in that message is
written by whoever connected. Those tests also disable proxy autodetection: the
hostnames are invented, and an ambient `HTTPS_PROXY` would send the request
somewhere other than the gateway under test.

Ports come from a `flock`-claimed band rather than a pid-derived one, so two
test binaries cannot collide however their process ids fall.

[agentgateway]: https://agentgateway.dev
[rusty_mcp]: https://github.com/baileyrd/rusty_mcp
[rusty_tls]: https://github.com/baileyrd/rusty_tls
[rusty_a2a]: https://github.com/baileyrd/rusty_a2a
