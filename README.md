# ScopeGuard

**Website security auditing from your terminal.** Runs in Windows CMD / PowerShell, Linux, and macOS with Python 3.10 or newer. No third-party runtime dependencies.

ScopeGuard performs real HTTP requests and certificate verification against the URLs you provide. It collects evidence, explains remediation, and produces terminal, JSON, standalone HTML, and Markdown reports. Use it only for websites you own or have permission to test.

This is a configuration auditor and a foundation for a pentesting project, **not a complete penetration testing suite**. It does not exploit vulnerabilities or claim that configuration warnings prove an application is exploitable.

## Quick start

Extract this project, open a terminal in the folder containing `pentest.py`, and run:

**Windows CMD or PowerShell**

```powershell
py -3 pentest.py demo --output reports/demo
py -3 pentest.py scan https://your-authorized-site.example --output reports/site
```

**Linux / macOS**

```bash
python3 pentest.py demo --output reports/demo
python3 pentest.py scan https://your-authorized-site.example --output reports/site
```

Replace the `.example` placeholder with your authorized target. The `demo` command uses synthetic data and makes no network requests. Open `reports/demo.html` in your browser to preview a report. There is no website server to run.

You can also run `python -m scopeguard` from this folder. If you want a `scopeguard` command on your PATH, install locally with `python -m pip install .` in a virtual environment. Pip may fetch build tools; direct execution above requires no installation or downloads.

## Checks included

| Area | What is inspected | Limits |
| --- | --- | --- |
| HTTP transport | Whether the final page uses HTTPS | No downgrade attacks or HSTS preload lookup |
| TLS | System trust and hostname verification, negotiated protocol/cipher, certificate expiration | One verified direct handshake; does not enumerate supported protocol versions |
| Response headers | HSTS, nosniff, enforced CSP, framing restrictions, explicit weak referrer policy | CSP review is heuristic, not a full policy validator; absence alone does not establish XSS |
| Cookies | Secure, HttpOnly, SameSite, and cookie prefix requirements | Cookie sensitivity is unknown; cookie values are not retained |
| HTML | HTTP resource references in HTTPS pages; password forms using HTTP or GET | No JavaScript rendering, form submission, browser execution, or authenticated session |
| Technology disclosure | Server / X-Powered-By headers | Informational; no version-to-CVE guesses |
| Optional CORS probe | Sends one GET with a unique `.invalid` Origin and checks the response | Unauthenticated; sensitive data access requires manual validation |

Findings use `high`, `medium`, `low`, or `info`, with evidence and a confidence label. There is no artificial overall risk score. A finding is an observed configuration or a candidate requiring manual validation, not a confirmed exploit.

## Commands

```bash
# Inspect the starting page only
python pentest.py scan https://your-authorized-site.example --max-pages 1 --depth 0

# Bounded same-origin crawling, with a slower request pace
python pentest.py scan https://your-authorized-site.example --max-pages 20 --depth 2 --delay 1

# Add the optional, low-impact CORS Origin probe
python pentest.py scan https://your-authorized-site.example --active

# Scan several explicitly listed targets sequentially
python pentest.py scan --targets targets.txt --output reports/engagement

# Fail a CI job on medium-or-higher findings
python pentest.py scan https://your-authorized-site.example --fail-on medium --quiet --no-color

# Explicitly use the operating environment's proxy settings
python pentest.py scan https://your-authorized-site.example --use-proxy

# A local development server with a private/self-signed certificate
python pentest.py scan https://localhost:8443 --insecure

# Help and version
python pentest.py scan --help
python pentest.py --version
```

For Windows, replace `python` with `py -3` if needed. Quote a URL containing `&`, particularly in CMD. In Windows PowerShell 5.1, view an exit status with `$LASTEXITCODE`; in CMD use `echo %ERRORLEVEL%`.

An input file contains one URL per line; blank lines and lines starting with `#` are ignored. Maximum: 20 entries, 64 KiB. See `targets.example.txt`.

### Boundaries and defaults

- Default: 5 pages, depth 1, 50 total HTTP requests per target, 10-second socket timeout, 0.4-second minimum request interval, and 512 KiB inspected per response.
- The request budget includes redirect hops and the CORS probe. Direct TLS inspection is one additional connection, outside the HTTP budget. A socket timeout is not a total scan deadline.
- Crawling stays on the starting origin (scheme, host, port). A standard same-host HTTP port 80 to HTTPS port 443 upgrade is allowed; no other host, port change, or HTTPS downgrade is followed. If a site redirects to `www`, start with the exact `www` URL explicitly.
- The crawler ignores URL fragments, discovered query-bearing links, common binary assets, and common action paths such as `/logout` and `/delete`. An explicitly supplied target URL may contain a query.
- It does not consult robots.txt. Crawl only authorized paths; GET requests can have application side effects even when no forms are submitted. Use `--max-pages 1 --depth 0` for a single-page review.
- At most five redirects are followed. HTTP 429 stops further requests for that target. Network failures, blocked redirects, and exhausted budgets are recorded as errors rather than being interpreted as clean results.
- No passwords, login, POST requests, credential attacks, port sweeps, exploit payloads, hidden-file guessing, or unsolicited third-party requests.
- Environment proxy variables are ignored unless `--use-proxy` is passed. With a proxy, the separate direct TLS check is skipped because it could observe a different network path. Configure your trusted inspection CA rather than disabling verification.
- TLS verification is enabled by default. `--insecure` only changes the HTTP client; the independent TLS check still verifies the certificate, and the report records the setting.
- Compressed response bodies are not decoded if a server disregards `Accept-Encoding: identity`; HTML inspection is partial. Truncated HTML may omit protections or forms beyond the size limit. Treat missing-policy observations accordingly.
- Static HTML parsing does not see JavaScript-created forms/resources, external stylesheet imports, every srcset variant, or browser-only behavior. CSP parsing is intentionally limited.

## Reports and exit codes

`--output reports/client` produces `reports/client.json`, `.html`, and `.md`. Without `--output`, a unique timestamped prefix is created in `reports/`. Multi-target runs append an index to an explicit output prefix. Existing reports are not replaced unless you pass `--overwrite`.

HTML reports are self-contained, responsive, printable, and work offline. They do not load fonts, scripts, or assets from third parties. JSON includes findings, inspected pages, scan bounds, TLS results, errors, partial checks, timestamps, and summary counts. `schema_version` is `1.0`.

Reports exclude query **values**, Set-Cookie values, and response bodies; query names, URL paths, cookie names, and selected header evidence remain. Paths or header evidence can still contain sensitive information. Store reports privately and review them before sharing. Terminal control characters are removed from remote text; HTML reports escape remote content.

| Exit code | Meaning |
| --- | --- |
| `0` | Command completed without recorded errors; no requested severity threshold was crossed |
| `1` | At least one finding meets `--fail-on` |
| `2` | Invalid input, I/O problem, or one or more scan errors; takes precedence over findings |
| `130` | Interrupted with Ctrl+C; a partial report for the current target is not saved |

`--fail-on none` is the default. A zero exit code does not mean that the website is secure. TLS, HTTP, or partial coverage problems are visible in the report. Severity gates include candidates requiring manual validation; review confidence labels.

## Development and tests

```bash
python -m unittest discover -s tests -v
python -m compileall -q scopeguard pentest.py
```

Tests run against controlled localhost fixtures and synthetic responses. They do not scan external websites. Certificate integration tests use the `openssl` command if available and otherwise skip only those tests. GitHub Actions runs the suite on Windows, Linux, and macOS with Python 3.10 and 3.13.

Project layout:

```text
scopeguard/
  cli.py          Command-line input and exit codes
  transport.py    Scoped, paced HTTP requests and bounded redirects
  checks.py       Header, cookie, HTML, and CORS rules
  scanner.py      Crawl orchestration and TLS inspection
  models.py       Versioned report model and evidence sanitization
  reporting.py    Terminal, JSON, HTML, and Markdown output
tests/            Local fixture and regression tests
pentest.py        Direct launcher
```

## References

Rule design is informed by the [OWASP HTTP Headers Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html), [OWASP CSP guidance](https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html), and [OWASP TLS guidance](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Security_Cheat_Sheet.html). This project is not an OWASP certification or complete OWASP testing implementation.

MIT licensed. See `LICENSE`.
