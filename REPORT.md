# Report: scrapy/scrapy, HttpProxyMiddleware crash on proxy URLs with only a user name

## Read this first: Scrapy auto-closes PRs it thinks an LLM opened unattended

`.github/workflows/close-spam-prs.yml` runs
[`scrapy/unattended-pr-guard`](https://github.com/scrapy/unattended-pr-guard)
(pinned at `2ea7eff`, v0.2.0) on every newly opened PR, and re-runs it every 6
hours on PRs from the last 3 days. Any one of these signals is enough for it to
label the PR and close it:

- **Agent branch:** the head branch matches
  `^(agent|codex|claude|cursor|devin|copilot|jules|bot)[/_-]` (case-insensitive,
  from `index.js`). **The fix branch, `claude/youthful-faraday-bzph1c`,
  matches.** A PR opened from it will be closed.
- **Rejection burst:** at least 1 PR by the author closed unmerged in a repo
  they don't own, in the last 30 days.
- **Spray breadth:** PRs opened against more than 2 repos the author doesn't
  own within a single week.
- **Assistant voice:** the author's recent public comments read like assistant
  output (headings, bullet lists, many em dashes, stock acknowledgement phrases).

Its README says the purpose is to close PRs "that an LLM is opening
unattended". I checked two recent bug-fix PRs, #8430 and #8424; both were
closed this way, and the author of #8424 was then blocked.

Nothing forbids AI-assisted PRs. `AGENTS.md` is written for agents, and
contributing.rst has no AI policy. So I did not stop. But the maintainers
plainly want a human attending each PR. My suggestion is to open this one only
if Wei-Lun will personally own it: read the diff, open it, and answer review
comments in their own words. Pushing the same commit under a branch name without
the agent prefix only removes one signal. It does not make an unattended PR
attended, which is what the guard exists to catch. Given how many repos this
account sends PRs to, the rejection-burst and spray-breadth signals may fire
anyway. I only pushed to the branch this session was assigned.

## Branches

- Fix branch: `claude/youthful-faraday-bzph1c` on `b84330808/scrapy`
- Fix commit: `f10e4b296cb7ba22dcc72df330cddbfb2f759674`
- Target base: `scrapy/scrapy` `master` at
  `51c10b14fca6d272587a5c23cf3ae8effdff16cd`. The fork's `master` was at the
  same commit when I branched.
- Commit author and committer: `b84330808 <b84330808@gmail.com>`, with trailer
  `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Files changed: `scrapy/downloadermiddlewares/httpproxy.py` (1 line) and
  `tests/test_downloadermiddleware_httpproxy.py` (adds a parametrized case to
  the existing empty-password test).

## The bug

`HttpProxyMiddleware._get_proxy()` passes the password from
`urllib.request._parse_proxy()` straight to `unquote()`. For a proxy URL with a
user name and no `:`, such as `http://user@proxy.example:3128`,
`_parse_proxy()` returns `None` as the password, so `unquote(None)` raises
`TypeError: argument of type 'NoneType' is not iterable`.

- If the value is in `http_proxy`/`https_proxy`, `HttpProxyMiddleware.__init__`
  raises, because it only catches `ValueError`. The crawler cannot be built, so
  every crawl fails while that variable is set.
- If the value is in `meta["proxy"]`, the request fails with the same error.

Upstream issue #1274 and PR #1313 (2015) fixed `user:@proxy` (empty password).
They changed `if user and password:` to `if user:`, but they only tested the
form with a colon. The form without a colon has crashed ever since.

The fix sends the user name with an empty password, `password or ""`. That
gives the same header as `user:@proxy`, and it matches how Scrapy handles user
info in request URLs: `UriUserInfoMiddleware` sets only `http_user`, and
`HttpAuthMiddleware` then defaults the password to `""`.

## PR title

```
Fix HttpProxyMiddleware crash on proxy URLs with only a user name
```

## PR body

The template's first line is `Resolves #NNNN.`. There is no issue to resolve,
so I left that line out. AGENTS.md says not to open an issue before a PR.
Leave the checkbox unticked unless it is true for the person opening the PR.

```markdown
`HttpProxyMiddleware` raises `TypeError` for a proxy URL with a user name but no password, such as `http://user@proxy.example:3128`, because `_parse_proxy()` returns `None` as its password; set in `http_proxy` or `https_proxy`, such a URL keeps the middleware from being built at all. This sends the user name with an empty password, as #1313 already does for `http://user:@proxy.example:3128`.

- [ ] I have read the [contributing guide](https://docs.scrapy.org/en/master/contributing.html) and [set up pre-commit](https://docs.scrapy.org/en/master/contributing.html#pre-commit).

I used Claude Code to write this change.
```

## Fail-before (fix reverted, new test kept)

I ran this with the fix stashed, using the tox `py` environment's interpreter.
`tox -e py -- <test>` gave the same 1 failed, 1 passed result.

```
$ .tox/py/bin/python -m pytest "tests/test_downloadermiddleware_httpproxy.py::TestHttpProxyMiddleware::test_proxy_auth_empty_passwd" -v --tb=line -p no:cacheprovider
============================= test session starts ==============================
platform linux -- Python 3.13.17, pytest-9.2.0, pluggy-1.7.0 -- /home/user/scrapy/.tox/py/bin/python
rootdir: /home/user/scrapy
configfile: pyproject.toml
plugins: platformdirs-4.14.0, rerunfailures-16.7, xdist-3.8.0, twisted-1.14.3, timeout-2.4.0, cov-7.1.0
collecting ... collected 2 items

tests/test_downloadermiddleware_httpproxy.py::TestHttpProxyMiddleware::test_proxy_auth_empty_passwd[https://user:@proxy:3128-https://username:@proxy:3128] PASSED [ 50%]
tests/test_downloadermiddleware_httpproxy.py::TestHttpProxyMiddleware::test_proxy_auth_empty_passwd[https://user@proxy:3128-https://username@proxy:3128] FAILED [100%]

=================================== FAILURES ===================================
E   TypeError: argument of type 'NoneType' is not iterable
/usr/lib/python3.13/urllib/parse.py:706: TypeError: argument of type 'NoneType' is not iterable
=========================== short test summary info ============================
FAILED tests/test_downloadermiddleware_httpproxy.py::TestHttpProxyMiddleware::test_proxy_auth_empty_passwd[https://user@proxy:3128-https://username@proxy:3128]
========================= 1 failed, 1 passed in 0.38s ==========================
```

The traceback from the tox run fails in `HttpProxyMiddleware.__init__`, which
means the environment-variable case breaks before any request is processed:

```
scrapy/downloadermiddlewares/httpproxy.py:28: in __init__
    self.proxies[type_] = self._get_proxy(url, type_)
scrapy/downloadermiddlewares/httpproxy.py:51: in _get_proxy
    creds = self._basic_auth_header(user, password) if user else None
scrapy/downloadermiddlewares/httpproxy.py:42: in _basic_auth_header
    user_pass = f"{unquote(username)}:{unquote(password)}".encode(
E       TypeError: argument of type 'NoneType' is not iterable
```

## Pass-after (with the fix)

```
$ .tox/py/bin/python -m pytest "tests/test_downloadermiddleware_httpproxy.py::TestHttpProxyMiddleware::test_proxy_auth_empty_passwd" -v --tb=line -p no:cacheprovider
============================= test session starts ==============================
platform linux -- Python 3.13.17, pytest-9.2.0, pluggy-1.7.0 -- /home/user/scrapy/.tox/py/bin/python
rootdir: /home/user/scrapy
configfile: pyproject.toml
plugins: platformdirs-4.14.0, rerunfailures-16.7, xdist-3.8.0, twisted-1.14.3, timeout-2.4.0, cov-7.1.0
collecting ... collected 2 items

tests/test_downloadermiddleware_httpproxy.py::TestHttpProxyMiddleware::test_proxy_auth_empty_passwd[https://user:@proxy:3128-https://username:@proxy:3128] PASSED [ 50%]
tests/test_downloadermiddleware_httpproxy.py::TestHttpProxyMiddleware::test_proxy_auth_empty_passwd[https://user@proxy:3128-https://username@proxy:3128] PASSED [100%]

============================== 2 passed in 0.32s ===============================
```

## Other checks

- `tox -e py -- tests/test_downloadermiddleware_httpproxy.py tests/test_downloadermiddleware_uriuserinfo.py tests/test_downloadermiddleware_httpauth.py`: 103 passed.
- `tox -e mypy`: "Success: no issues found in 529 source files".
- `tox -e pylint`: "Your code has been rated at 10.00/10".
- `pre-commit run --all-files`: ruff check, ruff format, blacken-docs,
  end-of-files, trailing whitespace, Sphinx Lint and sphinx-scrapy all passed.
  **zizmor failed.** It could not reach the GitHub API from this container
  (`401 Unauthorized` listing tags for `actions/checkout`). It only audits
  `.github/workflows`, which this change does not touch, but I could not run it
  to a pass here.
- I did not run the `docs` envs, because no docs or docstrings changed. I did
  not run a `min-` env, because the change depends only on the standard
  library.

## Duplicate search

The session's GitHub API access is limited to the fork, so I searched upstream
through the public github.com search pages. All searches cover open and closed
issues and PRs unless noted.

| Query | Results | Relevant? |
|---|---|---|
| `proxy without password` | 12 | No |
| `proxy empty password` | 7 | #1274 (closed issue) and #1313 (merged PR) cover `user:@proxy`, not `user@proxy`. #8427 is about `from_curl`. |
| `proxy username only` | 8 | No. #2526/#2530 are about reading credentials from `meta["proxy"]`. #5606 has no credentials in its URL. |
| `proxy TypeError` | 9 | No. #2127 and #4649 are about proxy URLs with no scheme. |
| `_basic_auth_header` | 1 | No (#3316, a closed refactor PR) |
| `proxy "NoneType" unquote` | 0 | n/a |
| `"argument of type 'NoneType' is not iterable"` | 1 | No (#833) |
| `is:pr HttpProxyMiddleware` | 25 | No. #7348 (closed) improves the error for a missing scheme. |
| `is:pr httpproxy password` | 14 | No |
| Newest 25 PRs and newest 25 issues | 50 | None are about proxies |

I found no issue or PR, in any state, for a proxy URL with a user name and no
password.

## Policy check

- `docs/contributing.rst`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md` and
  `README.rst` contain no AI or agent policy. Nothing forbids agent-made PRs,
  requires the author to attest to personal testing or review, or requires
  replies written without AI.
- `AGENTS.md` gives instructions for agents. The PR description should be one or
  two sentences on what and why, with no section headers, plus `Closes #N` for
  any issue it resolves. After a review, only add commits on top; don't rebase,
  amend or force-push.
- The PR template's checkbox asks the author to confirm they read the
  contributing guide and set up pre-commit. Tick it only if that is true.
- The unattended-PR guard is described at the top of this report.
- No repo file tried to make an agent do anything odd, such as adding markers,
  adding emoji, or stopping silently.

## Other bugs found but not fixed

These could be candidates for future sessions. I reproduced the first one
myself. The rest come from read-only helper agents that ran repro scripts; I
did not re-verify them.

1. **`scrapy genspider` with a host and port** (verified by me).
   `scrapy genspider s localhost:8000` writes `allowed_domains = [""]` and
   `start_urls = ["localhost:8000"]`, which has no scheme.
   `https://example.com:8080` gives `allowed_domains = ["example.com:8080"]`.
   Either way, `OffsiteMiddleware` filters every followed link, because it
   compares host names. Cause: `urlparse()` treats `host:` as a scheme in
   `verify_url_scheme()`, and `extract_domain()` returns `netloc`, not
   `hostname` (`scrapy/commands/genspider.py:34-44`). No upstream issue or PR
   found.
2. **`HTTPERROR_ALLOWED_CODES` given as a string** (for example
   `-s HTTPERROR_ALLOWED_CODES=404`) is never converted to `int`, so it matches
   nothing (`scrapy/spidermiddlewares/httperror.py:43-45`). The retry,
   redirect and httpcache code settings all convert.
3. **`RFC2616Policy` with a bare `Cache-Control: max-age`**: `int(None)` raises
   `TypeError`, which is not caught. The response is cached anyway, so later
   requests for that URL crash (`scrapy/extensions/httpcache.py:199-203`).
4. **An empty feed in `create` mode** over an existing file: `FileExistsError`
   escapes `FeedExporter._close_slot()`, instead of the logged conflict the
   `FEED_MODE` docs describe (`scrapy/extensions/feedexport.py:898-901`).
5. **Resuming a `JOBDIR` crawl with `batch_item_count` > 1** rewrites the last
   batch file of the previous run (`scrapy/extensions/feedexport.py:818-826`).
6. **An `Item` subclass that sets an inherited `Field` to a non-Field** (such
   as `price = None`) leaves `fields["price"] = None`, which makes exporters
   crash (`scrapy/item.py:57-59`).
7. **`Request.to_curl()`** collapses repeated whitespace and newlines inside
   the body and header values, and does not escape `'`
   (`scrapy/utils/request.py:281-282`).

Upstream is actively working on `from_curl` (#8425, #8426, #8427, #8433) and
on robots.txt and sitemaps (#8418, #8419, #8434, #8435), so avoid those areas.
