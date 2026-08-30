# test-monitor

Throwaway repo for testing [rl-protect-monitor](https://github.com/kburich/rl-protect-monitor)
end-to-end against real GitHub — issues, labels, orphan baseline branch,
notification emails — rather than against unit fixtures.

## The fixture

`package-lock.json` pins five deliberately outdated, zero-dependency packages,
each carrying published CVEs in the pinned version. They are **never
installed**: the monitor reads the lockfile and looks the packages up in the
Spectra Assure database, so nothing here is fetched or executed. Zero-dependency
packages keep the scan small, and scans are metered in entitlement units.

| Package | Pinned | Why |
|---|---|---|
| `lodash` | 4.17.11 | prototype pollution (CVE-2019-10744, CVE-2020-8203) |
| `minimist` | 1.2.0 | argument injection (CVE-2020-7598, CVE-2021-44906) |
| `moment` | 2.29.1 | path traversal / ReDoS (CVE-2022-24785, CVE-2022-31129) |
| `node-fetch` | 2.6.0 | header leak on redirect (CVE-2020-15168) |
| `ua-parser-js` | 0.7.28 | ReDoS (CVE-2022-25927) — benign version |

## Running it

`RL_TOKEN` must be set as a repo secret. While a staircase is being walked the
workflow runs on every push to `main` and the cron is commented out; once it
passes, drop the push trigger and re-enable the cron.

The workflow pins the action at the branch under test, not `@v2` — see the
comment at the top of `.github/workflows/rl-protect-monitor.yml` for why.

## The staircase (append-only issues)

The rolling issue is an append-only alert log: its body is written once, at
creation, with that run's delta, and every later delta — resolutions included
— is a comment. Nothing is ever edited. Each step exercises something the unit
tests cannot:

1. **Migration.** Push with nothing else changed, against a schema-2 baseline
   from the previous design. Expect exactly one baseline commit (schema 3, the
   `stats` block and `alerted` flags gone), no comment on the open issue, and
   its body untouched.
2. **Quiet run.** Push a no-op. Expect no baseline commit and no comment.
3. **New delta on an existing issue.** Strip a few finding records out of
   `rl-protect-baseline/<id>:.rl-protect/baseline.json`, push there, then
   push a no-op to `main`. Those findings read as new: a comment lands on the
   open issue and the body — still the old stats page — is not edited.
4. **Resolution comment.** Remove a package from the lockfile and push.
   Expect a comment headlined `N resolved` with a `Resolved findings` table,
   no 🚨, and — again — no body edit.
5. **Resolution with no open issue.** Close the issue, remove another package
   from the lockfile, push. Expect *nothing*: no issue opened, no comment.
6. **Fresh issue.** Strip findings from the baseline again and push a no-op.
   Expect a new issue whose body *is* the delta comment rendering, `kburich`
   assigned, and no follow-up comment on it.
7. **The wrapper.** Swap to the reusable workflow, dispatch once, confirm the
   inputs still thread through.

## First-run alerting

`alert-on-first-run` needs a monitor with no baseline. Rather than deleting
the real one, give the run a throwaway `monitor-id` (say `first-run-test`)
with `alert-on-first-run: "true"`: it gets its own baseline branch and its
own rolling issues, so it is a genuine first run beside the real monitor, and
the whole backlog lands as the new issue's body. Afterwards drop the override,
delete `rl-protect-baseline/<id>` and close the orphaned issue by hand.

## The malware path

Stripping the baseline only promotes findings already present in the scan, so
a `🚨` alert needs a package the database actually flags. The fixture pins
`ua-parser-js` at `0.7.28` for exactly this: `0.7.29` is the hijacked release,
flagged as both `malware` and `tampering`. Bump to it and push — a `🚨` issue
opens in the critical bucket (and `CVE-2021-4229`, the CVE for the hijack
itself, lands as a genuinely new finding in the standard one). Revert and push
— both issues get a resolution comment, the malware one without the siren.
Nothing is ever installed; the lockfile only names the version.
