# Freeborn Live Audit

**Audit date:** 2026-11-08 (session evidence only)  
**Scope:** `spencerbby8-gif/freeborn-original`, Vercel project `freeborn`, Supabase project `Freeborn`. `freeborn-web` was not queried.

## Executive summary

The three selected systems are the correct ones and are connected. GitHub `freeborn-original` is the Vercel Git source. Vercel production is `https://freeborn-umber.vercel.app`. Supabase Auth `site_url` and the redirect allow-list point at that host plus Vercel preview hosts and localhost.

The delivery path is active: a GitHub push or pull request creates a Vercel deployment. Production is **not** serving `main`; it serves commit `a14387cf` from `fix/remediation-2026-11-07` (PR #33), READY and promoted. `main` is `cbc1e8c2` (2026-11-02). The newest preview is from unsigned commit `91d6a767` on `remediation/clean-replay-20261108`.

All 41 public Supabase tables have RLS enabled and forced. `anon` has no public-table grants. Some sensitive tables retain broad `authenticated` DML grants without a matching policy returned in this session; forced RLS should deny policy-less operations, but those grants are defense-in-depth debt. Two active Edge Functions have JWT verification disabled. Auth is email-only, signup is enabled, minimum password length is 6, and HIBP checks are off.

No service-role key was found in the reviewed committed source. Client exposure is limited to expected `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY` names. Secret values were not read.

## Confirmed inventory and map

| System | Confirmed identifier | Status |
|---|---|---|
| GitHub | `spencerbby8-gif/freeborn-original`, id `1407749219`, public, default `main` | Verified |
| Vercel | `freeborn`, id `prj_3Y6fEI3YedWsffXGHXWPwesu7BOL`, hobby plan | Verified |
| Supabase | `Freeborn`, ref `ihwtpwlszegrjgakqsxx`, `us-east-1`, `ACTIVE_HEALTHY`, Postgres 17.6.1.141 | Verified |

```text
GitHub freeborn-original (repoId 1407749219)
  -> Vercel freeborn: GitHub link, productionBranch=main,
     createDeployments=enabled, deployHooks=[]
  -> Supabase Freeborn: Auth site URL is freeborn-umber.vercel.app
```

**Interpretation:** this Supabase project is the Vercel backend because its Auth URL matches the Vercel domain and Vercel holds Supabase URL/key environment-variable names. Exact encrypted Vercel values were not decrypted.

## Verified pipeline

| Stage | Evidence |
|---|---|
| Source and branch control | Default `main` is `cbc1e8c2`. `main` is the only protected returned branch; required status check is `verify`, enforced for non-admins. |
| CI | `.github/workflows/ci.yml` triggers on push/PR to `main` and manual dispatch. It runs Node 24, `npm ci`, secret scan, typecheck, lint, trust harness, and production build. Latest listed CI on `main`, run `41410140082`, succeeded. |
| Build | Vercel link: GitHub repo `freeborn-original`, repo ID `1407749219`, production branch `main`. Framework `nextjs`; Node `24.x`; turbopack; region `iad1`; fluid functions; default timeout 300 seconds. |
| Production deployment | `dpl_8o5rr75oibJqMnBqJNzTNpXGAx52`, READY/PROMOTED, SHA `a14387cf`, branch `fix/remediation-2026-11-07`, `githubCommitVerification=unverified`. |
| Preview deployment | `dpl_7msu6Csii8kguGL6ksU8m5J3zF23`, READY/STAGED, SHA `91d6a767`, branch `remediation/clean-replay-20261108`, also unverified. |
| Runtime integration | `src/lib/supabase/{browser,server,proxy,api-auth}.ts` exist; Supabase Auth points to the Vercel production host; DB host is `db.ihwtpwlszegrjgakqsxx.supabase.co`. |

### Authentication boundaries

- GitHub to Vercel uses a stored Vercel Git credential; credential material was inaccessible.
- Vercel to Supabase uses encrypted/sensitive env variables. Vercel OIDC is enabled (`issuerMode=team`). No Vercel env named `SERVICE_ROLE` was returned.
- Edge function secret metadata includes `SUPABASE_SERVICE_ROLE_KEY` and `SUPABASE_DB_URL`; values were not exposed.

## Findings

| ID | System | Finding | Severity | Evidence | Confidence |
|---|---|---|---|---|---|
| F-01 | Vercel/GitHub | Production is not `main`; it serves `a14387cf` from `fix/remediation-2026-11-07`, while `main` is `cbc1e8c2`. | High | Vercel production metadata; branch list | Verified |
| F-02 | Vercel | Production and current preview commits are unverified. | Medium | `githubCommitVerification=unverified` | Verified |
| F-03 | GitHub | `main` protection requires check `verify` for non-admins. Required-review count was not returned. | Info | Branch protection object | Partially Verified |
| F-04 | GitHub | CodeQL is visible on an unmerged remediation branch; `main` workflow directory returned only `ci.yml`. A Dependabot run failed. | Medium | Workflow runs and source listing | Verified |
| F-05 | GitHub | PRs #33, #32, #31 (draft), and #23 remain open; #33's commit is already serving production. | High | Open PR list and Vercel metadata | Verified |
| F-06 | Vercel | Returned public env names are expected Supabase URL/anon-key/app URL names; no service-role name returned. | Info | Vercel env key list | Verified |
| F-07 | Vercel | Preview omits several production-only engine/admin variables and uses separately scoped Supabase variables. | Medium | Vercel env targets | Verified |
| F-08 | Vercel | `autoExposeSystemEnvs=true`. | Low | Project config | Verified |
| F-09 | Vercel | Sole domain is verified `freeborn-umber.vercel.app`; no custom domain or redirect. | Info | Domain list | Verified |
| F-10 | Supabase | RLS is enabled and forced on all 41 returned public tables. | Info | `pg_class` query | Verified |
| F-11 | Supabase | `anon` has no public-table grants. | Info | Role-grants query | Verified |
| F-12 | Supabase | `authenticated` retains broad DML grants on sensitive policy-less tables. Forced RLS should deny policy-less operations, but grants are broader than visible policies. | High | Role grants compared with `pg_policies` | Partially Verified |
| F-13 | Supabase | `chat_signals` grants authenticated full DML, but only a SELECT policy was returned. | Medium | Grants and policy query | Verified |
| F-14 | Supabase | `verification_audit_logs` grants authenticated INSERT and SELECT; predicate SQL was not retrieved. | Medium | Grants and policy names | Partially Verified |
| F-15 | Supabase | `profile-photos` is public, 8MB-limited, and allows jpeg/png/webp. `verification-selfies` and `chat-media` are private; no chat-media policy returned. | High | Bucket and Storage-policy lists | Partially Verified |
| F-16 | Supabase | `fb14-storage-cleanup` and `fb15-selfie-retention` are ACTIVE with `verify_jwt=false`. | High | Edge-function list | Verified |
| F-17 | Supabase | Email-only auth; signup enabled; password minimum 6; HIBP/captcha off; refresh rotation on; email confirmation required. | Medium | Auth config | Verified |
| F-18 | Supabase | TOTP enrollment/verification is enabled; app-side requirement was not proven. | Low | Auth config | Partially Verified |
| F-19 | Supabase | Legacy `anon` and `service_role` keys coexist with publishable and secret keys. Values not read. | Info | Key metadata | Verified |
| F-20 | GitHub | `.env.example` includes real Supabase host plus placeholder keys, not committed credentials. | Low | `.env.example` | Verified |
| F-21 | GitHub | CI maps GitHub Supabase secrets into `NEXT_PUBLIC_*` production-build vars; Actions-secret existence was not checked. | Info | `ci.yml` | Partially Verified |
| F-22 | Supabase | Edge-function secret names include service-role and DB URL secrets; neither appears as a Vercel env name. | Info | Supabase secret metadata; Vercel env names | Verified |

## Remediation actions

1. Reconcile the production source: merge/review the serving SHA into `main`, or explicitly make the intended branch the Vercel production branch.
2. Require signed commits for deployable changes.
3. Remove unnecessary `authenticated` DML grants from policy-less sensitive tables; grant only what explicit RLS policies require.
4. Retrieve and review `USING` and `WITH CHECK` SQL for all database and Storage policies.
5. Review JWT-disabled workers. Retain `verify_jwt=false` only with a strong, separate, failing-closed cron/shared-secret check.
6. Raise the Auth password floor, enable HIBP checks, and enable CAPTCHA before broad signup operation.
7. Merge CodeQL to `main` and investigate the failed Dependabot run.
8. Declare the intended preview behavior for production-only engine/admin variables.

## Unverifiable — access required

| Item | Required access |
|---|---|
| Exact Supabase URL value in Vercel env | Decrypted Vercel env read or safe runtime host diagnostic |
| Vercel build/function logs | Deployment log API/dashboard access |
| RLS predicate text | Query `pg_policies.qual` and `with_check` |
| Actual authenticated write outcomes | Disposable Auth users and controlled PostgREST tests |
| Edge-function source and secret checks | Function-body read for both slugs |
| GitHub Actions secret metadata/webhooks | Secret listing and webhook-admin listing |
| Full branch review protection | Full GitHub branch-protection response |
| Complete secret scan | Run `npm run check:secrets` or repository-wide search |
| DB foreign keys and indexes | Referential-constraint and `pg_indexes` queries |
| Network restrictions/custom hostname | Supabase admin endpoints |
| Runtime call proof | Controlled authenticated production trace |

Every verified claim above was supported by source, API, or read-only SQL evidence obtained during the audit session. Items not directly observed are explicitly marked as partial or unverifiable.
