# Design: `download_url` host-allowlist (#239 follow-up)

**Status:** built + gate-3 cleared (impl-validator COMPLETE · adversarial APPROVE-WITH-NITS · security SAFE). Port-forcing test added post-review. Ready to merge.
**Scope:** one contained security hardening in `GitHubSkillParserService`.
**Component:** api

## Problem

`GitHubSkillParserService#fetch_raw_content` fetches skill file bytes from the
`download_url` field of a parsed GitHub Contents API JSON response:

```ruby
def fetch_raw_content(url)
  uri = URI.parse(url)
  http = Net::HTTP.new(uri.host, uri.port)  # host + port from the URL, no check
  http.use_ssl = true                       # forced, but scheme never validated
  ...
end
```

`entry["download_url"]` is untrusted input (a field read verbatim from an
external response), yet the host, scheme, and port are taken from it with no
validation. `use_ssl = true` is hardcoded but `uri.port` is honored. This is the
deferred hardening flagged when the TOCTOU SHA-pin shipped (`a31bc017`).

## Threat model (gate-1, condensed)

| # | Threat | Severity |
|---|--------|----------|
| T1 | SSRF: a steered host makes the GET hit `169.254.169.254` / `localhost` / internal TLS, and **the response body is stored as `Resource.content`** — an SSRF-to-datastore exfil channel. Realistic trigger is a MITM defeating CA validation or future GitHub response-shape drift (a repo owner cannot set `download_url`; symlinks/submodules are filtered by `type=="file"`). **Defense-in-depth (Medium), not a live Critical.** | Medium |
| T2 | Port abuse: `uri.port` honored, so `:8443` reaches an internal TLS service on a non-443 port. | Medium |
| T3 | Redirect-follow regression: `Net::HTTP#request` does not auto-follow today; a future "improvement" that follows would bypass the allowlist. Must be a documented invariant. | Low (latent) |
| T4 | Unbounded body: no independent size cap beyond GitHub's `entry["size"]`. Low while host is pinned. | Note-only |

## Design

A single fail-closed choke point in `fetch_raw_content` (its only caller is
`fetch_and_parse_file`), validating **before any request is issued**.

### 1. Shared host constant

```ruby
# GitHubSkillParserService
RAW_CONTENT_HOST = "raw.githubusercontent.com"
HTTPS_PORT = 443
```

`external_skill_sync_service.rb:236` currently hardcodes the same host in the
stored-provenance URL string. Reference the constant there
(`GitHubSkillParserService::RAW_CONTENT_HOST`) so the two can't drift. No new
coupling — the sync service already instantiates the parser (line 49). The
provenance string is otherwise safe-by-construction (owner_repo regex-pinned,
40-hex SHA, `safe_repo_relative_path?`) and is **not** re-fetched here — no
behavioral change, just DRY.

### 2. Typed error + verified-URI guard

```ruby
class UnsafeDownloadUrlError < GitHubApiError; end

# Validate a Contents-API download_url BEFORE any request is issued. Fail closed:
# only an https URL whose host is EXACTLY raw.githubusercontent.com is allowed
# (case-insensitive, no suffix match — raw.githubusercontent.com.evil.com fails).
# The caller forces port 443 and never follows redirects. A poisoned-resolver /
# resolved-addresses-public? guard (as in SlackWebhookNotifier) is DEFERRED — see
# "Deferred" below.
def verified_download_uri(url)
  uri = URI.parse(url.to_s)
  unless uri.is_a?(URI::HTTPS) && uri.host&.downcase == RAW_CONTENT_HOST
    raise UnsafeDownloadUrlError,
      "Refusing skill download from disallowed host: #{uri.host.inspect} (only #{RAW_CONTENT_HOST} over https)"
  end
  uri
rescue URI::InvalidURIError
  raise UnsafeDownloadUrlError, "Malformed download_url"
end
```

### 3. Rewired `fetch_raw_content`

```ruby
def fetch_raw_content(url)
  uri = verified_download_uri(url)
  http = Net::HTTP.new(uri.host, HTTPS_PORT)  # force 443 regardless of uri.port
  http.use_ssl = true
  http.open_timeout = REQUEST_TIMEOUT
  http.read_timeout = REQUEST_TIMEOUT

  request = Net::HTTP::Get.new(uri)
  response = http.request(request)  # never follows redirects — any 3xx falls to
                                    # the else branch below as a hard failure.
  # ... unchanged 200 / 403 / else handling ...
end
```

- **Scheme:** rejected unless `URI::HTTPS` (explicit, not by handshake failure).
- **Host:** case-insensitive exact `==` against the allowlist literal. **Only
  `raw.githubusercontent.com`** — NOT `objects./media./codeload.` (those are
  LFS/media redirect targets and tarball hosts, never a Contents-API file
  `download_url`; adding them widens SSRF surface for zero ingestion benefit).
- **Port:** forced to 443 (mirrors `SlackWebhookNotifier`); `uri.port` ignored.
- **Redirects:** never followed; any 3xx is a hard failure (documented invariant).
- **Order:** validate before `http.request` → no request ever issued to a
  disallowed host, so no internal bytes are ever read (neutralizes T1's exfil).

### 4. Fail-closed behavior

`UnsafeDownloadUrlError < GitHubApiError` raised from `fetch_raw_content` is
caught by the existing per-entry `rescue => e` in `fetch_skills` (line 126),
recorded into `errors[]`, and surfaced via `source.record_error!`. **The file is
skipped with a recorded, diagnosable error — never silently dropped** (a
legitimate GitHub host drift must produce a signal, not a silent zero-ingest).

### Logging

`download_url` / host is a public raw URL, **not** a secret — safe to include the
offending host in the error for diagnosability. No response body is ever fetched
for a disallowed host (validate-before-request), so there is no internal-body
leak risk. Never switch to fetch-then-validate.

## Deferred (documented, not built here)

- **Poisoned-resolver / `resolved_addresses_public?` guard** (as in
  `SlackWebhookNotifier`): DEFERRED. The host is a fixed public literal and
  transport is TLS with standard CA verification — a resolver mapping the host
  to an internal IP cannot complete a valid TLS handshake (cert mismatch), and
  DNS-rebinding is N/A (host not attacker-controlled). Adding it would pull
  `resolv`/`ipaddr` + a forbidden-range set into a service with no DNS
  dependency, add latency, and add a new fail-closed edge (resolution hiccup
  blocks legit ingestion). Conscious divergence from the Slack pattern's
  rationale, recorded here.
- **CLI-side validation of the stored provenance `download_url`** (the CLI later
  downloads it): separate follow-up ticket. The stored value is
  safe-by-construction; low risk. Out of scope.

## Tests (api)

New cases in `github_skill_parser_service_test.rb`:

1. Cross-host `download_url` (e.g. `http://169.254.169.254/latest/meta-data/`)
   → file skipped, error recorded, **`assert_not_requested`** the internal host.
2. Suffix-match host `https://raw.githubusercontent.com.evil.com/x.md` → refused,
   no request to evil host.
3. Non-https (`http://raw.githubusercontent.com/x.md`) → refused.
4. Malformed `download_url` → refused, recorded error.
5. Regression: the existing happy-path suite (all `raw.githubusercontent.com`)
   stays green — no positive test changes needed.

## Gates

- `PARALLEL_WORKERS=1 bin/rails test test/services/github_skill_parser_service_test.rb`
  on the isolated `crewkit_test_wt1` DB.
- `test/sentinels/` + `rubocop`. No route/response-shape change → no OpenAPI.
