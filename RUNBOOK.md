# Oberon Logistics web runbook

Repo: [oberonlogistics/oberonlogistics-web](https://github.com/oberonlogistics/oberonlogistics-web)

Firebase project: `oberonlogistics-web` (Hosting only)

Production:

- https://oberonlogistics.com
- https://www.oberonlogistics.com
- https://oberonlogistics-web.web.app

Incident owner: Ollie (PM, Oberon Logistics).

## Deploy

A merge to `main` runs [`.github/workflows/firebase-hosting-merge.yml`](.github/workflows/firebase-hosting-merge.yml). That Action deploys Hosting to the live channel. It does not deploy anything else.

A pull request opened in this repo gets a preview channel from [`.github/workflows/firebase-hosting-pull-request.yml`](.github/workflows/firebase-hosting-pull-request.yml). The workflow skips a pull request whose head branch is on a fork.

Merge check, on the current head SHA:

1. PM sign-off and CTO sign-off on that SHA.
2. CI green on that SHA.

No manual `firebase deploy`.

## Health check

After a merge, confirm `firebase-hosting-merge.yml` is green for the merged commit:

```bash
gh run list \
  --repo oberonlogistics/oberonlogistics-web \
  --workflow firebase-hosting-merge.yml \
  --commit "$SHA" \
  --limit 5
```

Then require HTTP 200 and these title substrings:

```bash
set -euo pipefail

check() {
  local url="$1" needle="$2" tmp code
  tmp=$(mktemp)
  code=$(curl -sS --max-time 20 -o "$tmp" -w '%{http_code}' "$url")
  if [[ "$code" != "200" ]]; then
    echo "FAIL $url HTTP $code" >&2
    rm -f "$tmp"
    exit 1
  fi
  if ! grep -F -q "$needle" "$tmp"; then
    echo "FAIL $url missing: $needle" >&2
    rm -f "$tmp"
    exit 1
  fi
  echo "OK $url"
  rm -f "$tmp"
}

check "https://oberonlogistics.com/" "Landstar BCO - Oberon Logistics"
check "https://oberonlogistics.com/jobs.html" "Careers | Oberon Logistics"
check "https://oberonlogistics.com/carriers/" "Oberon Logistics - Carriers"

# After /jobs/ ships, also require 200 and the job title, and 301 from /jobs to /jobs/.
# JOB_TITLE="the shipped job title"
# check "https://oberonlogistics.com/jobs/" "$JOB_TITLE"
# curl -sS --max-time 20 -o /dev/null -w '%{http_code} %{redirect_url}\n' \
#   "https://oberonlogistics.com/jobs"
# expect: 301 https://oberonlogistics.com/jobs/
```

## Rollback

### Standard

Open a revert pull request and send it through the merge check. Never push directly to `main`.

```bash
git fetch upstream
git switch -c revert/hosting-SHA upstream/main
git revert --no-edit SHA
# merge commit:
# git revert --no-edit -m 1 MERGE_SHA
git push -u origin HEAD
```

Open that branch against `oberonlogistics/oberonlogistics-web` `main`. PM and CTO sign off on the revert head SHA. CI must be green. After the merge, the Hosting Action redeploys. Re-run the health check.

### Emergency

Use this only when production is broken and the revert pull request cannot wait. In the Firebase console, roll Hosting for project `oberonlogistics-web` back to the last good release. Then immediately open the revert pull request above so `main` matches live.
