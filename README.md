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

Manual dispatch only; the cron in the workflow is commented out until the
staircase below passes. `RL_TOKEN` must be set as a repo secret.

The workflow pins the action at `@main`, not `@v2` — see the comment at the top
of `.github/workflows/rl-protect-monitor.yml` for why.

## The staircase

Each step exercises something the unit tests cannot:

0. **Dry run.** Dispatch from a non-default branch. The action scopes that to a
   dry run: scan, job summary and delta artifact, but no baseline commit and
   nothing posted. Confirms token wiring and manifest auto-detection before
   anything durable is written.
1. **First run on `main`.** Expect the `rl-protect-baseline/<monitor-id>` orphan
   branch to be created, no issue, and a job summary reporting the pre-existing
   backlog.
2. **Immediate re-dispatch, nothing changed.** Expect a no-op, and *no* new
   baseline commit — the findings payload is identical.
3. **Force a new delta.** Strip a few finding records out of
   `rl-protect-baseline/<id>:.rl-protect/baseline.json`, push, dispatch. Those
   findings now read as new: the rolling issue opens, the delta lands as a
   collapsible comment, and the cumulative stats start counting. Subscribe to
   the issue first and read the actual email.
4. **Resolution path.** Remove an alerted package from the lockfile and
   dispatch — `lodash` was used for this. Resolving via the lockfile rather
   than by injecting a synthetic baseline record matters: a synthetic record
   was never alerted on, so it exercises neither the `resolved` counter nor
   the `outstanding` arithmetic. Expect the issue body to re-render with
   updated counters, `runs with alerts` *not* incremented, and *no* comment.
5. **The wrapper.** Swap to the reusable workflow, dispatch once, confirm the
   inputs still thread through.

The malware path cannot be manufactured this way — stripping the baseline only
promotes findings already present in the scan, so a `🚨` alert needs a package
the database actually flags.
