# github-core-clone-04

A standalone GitHub-compatible forge in a single Python file (stdlib only,
Python 3.13). One process answers the whole surface the contest scores:

- **git smart HTTP** — `git-upload-pack` v2 (ls-refs, fetch, shallow and
  deepen, plain packs) and v0 advertisements, plus `git-receive-pack` v0
  with fast-forward-only ref updates and sideband `unpack ok` / per-ref
  `ok` / `ng` reporting.
- **REST v3** — repositories, branches, commits, contents (create/update/
  delete with blob-sha checks), labels, issues, comments, pull requests,
  files, squash merges with three-way conflict detection, pagination via
  `Link: rel="next"`, and a non-empty `X-GitHub-Request-Id` on every reply.
- **GraphQL v4** — viewer/repository/issue/pullRequest/node, `__type`
  introspection, batched operations, and HTTP-200 errors that name the
  invalid field.
- **MCP** over streamable HTTP — `server/discover` for the 2026-07-28 era
  with a legacy `initialize` fallback, and eight tools (`create_repo`,
  `get_repo`, `list_repos`, `create_issue`, `get_issue`, `list_issues`,
  `create_pull_request`, `merge_pull_request`) on the same store as REST.

## What this variant optimizes

- **Raw-socket HTTP/1.1 server** (one daemon thread per connection, no
  pool cap): no per-request Date header, no `BaseHTTPRequestHandler`
  machinery, one `sendall` per reply, `TCP_NODELAY` on every socket.
  A bounded thread pool was measured to stall the 50-concurrent-clone
  workload at ~4 s (threads blocked in `recv` on idle keep-alive
  connections starve new ones until the client closes them); unbounded
  thread-per-connection fixed it (~98 ms p50).
- **No global request lock**: mutations take a short lock, reads run free,
  so concurrent clones actually run in parallel (zlib/sha1 release the
  GIL while they work).
- **Cached serialized issue bodies** per repo, invalidated on PATCH:
  the read workload hammers one issue 2,000 times.
- **Pack cache shared by v2 and v0 fetches**, built once per key behind
  a single build lock, so concurrent identical clones share one build.
- **Faster delta resolution**: REF_DELTA bases resolve through a dict
  (not a linear scan), the depth-limited fetch walk uses a deque with
  memoized tree closures, and delta application is a tight slice loop.

## Local scores

Kit conformance suite (practice trial-0, local runner), three runs:

| run | score | feature | agent | perf | push | clone | read | write |
|-----|-------|---------|-------|------|------|-------|------|-------|
| 1   | 948   | 600     | 200   | 148  | 50   | 0     | 50   | 48    |
| 2   | 984   | 600     | 200   | 184  | 50   | 43    | 49   | 42    |
| 3   | 990   | 600     | 200   | 190  | 50   | 43    | 50   | 47    |

120/120 conformance checks in every run; the state export is
byte-identical across runs (deterministic replay). Run 1 predates the
thread-per-connection fix (clone 0 points); runs 2–3 are the shipped
server. Per-workload p50s vary with VM load (read 608–738 ms, write
100–135 ms, clone 98 ms, push 248–282 ms).

## Run

```sh
python3 main.py   # needs MAKEFUN_ENV_URL (and MAKEFUN_SHOP_TOKEN)
```

Lifecycle: bind an ephemeral port, `POST $MAKEFUN_ENV_URL/register`
with `{"port": n}`, poll `GET $MAKEFUN_ENV_URL/done` until `{"done": true}`,
exit 0.

Determinism: every timestamp comes from `MAKEFUN_TRIAL_NOW` (or a fixed
base) plus a monotonic per-commit counter; request ids are counter-based;
no wall clock, no randomness anywhere in a response. The viewer login is
derived from the contest's fixed scenario seed with the same FNV-1a /
mix64 / xorshift64* stream the kit's wire contract uses — no wall clock,
no randomness in any response.
