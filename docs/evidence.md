# Evidence

What was actually measured, how, and what it does not show. Dates are November 2026.

## 1. A product-specific check against Nmap, on one CVE

**Setup.** One authorized test host on a private network, running Jenkins from the official WAR (2.441, then
2.442; SHA-256 verified against the published hashes) on a user-space JRE. Jenkins 2.441 is affected by
CVE-2024-23897; 2.442 contains the fix. The scanner ran the signed open catalog with a common-ports profile
(54 ports). Nmap 7.98 ran with service detection (`-sV`) and, separately, the `vuln and safe` script category,
over the same ports.

**Results.**

| | Nmap | SIGEA |
|---|---|---|
| Open ports | 22 (SSH), 8080 | 22, 8080 |
| Product on 8080 | "Jetty 10.0.18" | Jenkins 2.441 (from its own response header) and Jetty 10.0.18 |
| Mentions Jenkins or CVE-2024-23897 | No | Yes: `probable` (version older than 2.442 weekly or 2.426.3 LTS) |
| CVE entries listed | 27, all for Jetty, matched on the banner | One, for the product that matters |
| After upgrading Jenkins to 2.442 | Same output | `refuted` (the version is outside the vulnerable range) |

**Reading.** Jenkins is packaged on Jetty, so a banner-based tool names the server underneath. A check that
reads the product's own version identifies the product and can tell the fixed release from the vulnerable one.
The Jetty list is a set of hypotheses: it did not change when Jenkins was upgraded.

**What this does not show.** One product, one CVE, one host. The SIGEA finding is `probable`, not
`confirmed`: nothing probed the service's behavior, and a distribution could carry a fix while keeping the
version. This is not a general comparison of the two tools, and Nmap is not designed for this job.

## 2. Defects that only real use found

Running against a real host, and then installing from the package on a clean state, exposed four defects that
unit tests had not. Each one is now covered by a test.

| Found by | Symptom | Cause | Change |
|---|---|---|---|
| Scanning a real host | 36 checks answered `inconclusive` with no explanation | The connection budget (10) was spent by the port probes alone, so the HTTP checks were refused | The budget is sized automatically from the endpoints; a spent budget marks the report incomplete and fails the command |
| End-to-end test of the Jenkins flow | A present but empty `X-Jenkins` header was read as "no Jenkins", and the CVE rule answered `no_finding` | The marker required a numeric version instead of the header's presence | The marker is the header's presence; an unreadable version yields `inconclusive` |
| Installing from the package | Every login answered an error | The database role of the container deployment lacked privileges on tables added by later migrations | The privileges are now rendered from one embedded file (`grants`), used by the deployment, the docs and the tests |
| Installing from the package | A report could not be sent from the host | The agent identity names the server by its container address | An upload action that runs inside the project's network |

## 3. Installation rehearsals

The whole flow (build the package, initialize, migrate, create the administrator, start the API and worker
over TLS, enroll an agent, scan a real host with the signed catalog, upload the report, read it back) was run
twice from the package: with Docker Compose, and natively with the packaged control tool on a disposable
PostgreSQL, validating generated systemd user units without enabling them. Both were run on the maintainer's
machine, with an AI coding assistant executing the commands under the maintainer's direction. **No one outside the project has installed it yet**, which is the acceptance criterion for
the first release.
