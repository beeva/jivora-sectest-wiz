# Jivora — security platform test bed · Wiz

> ⚠️ **Intentionally vulnerable.** This repository exists solely to evaluate
> security scanning platforms. Do not deploy it, do not reuse its code, and do
> not treat any credential in it as real.

**Platform under test:** [Wiz](https://www.wiz.io) — specifically **Wiz Code**,
the code and IaC arm of the CNAPP.

## Read this before you score it

Wiz is a cloud-native application protection platform. Its actual value is a
connected cloud account: agentless workload scanning, runtime posture,
identity graphs, and the attack-path analysis that made its name. **A git
repository can supply none of that.**

Jivora today is Docker Compose on a single host with no cloud account to
connect. That mismatch is not a flaw in this test bed — it *is* the finding.
The honest question this repository answers is narrower than the other four:

> If Jivora migrated to AWS/EKS, is Wiz the platform it would want — and in the
> meantime, is Wiz Code alone competitive with the others on code scanning?

`infra/terraform/` and `infra/k8s/` describe exactly the deployment Jivora would
have *after* that migration. Scan them as a proxy for the post-migration world.

## Enabling it

1. Wiz requires a sales-led trial; there is no self-serve signup.
2. Create a service account in the Wiz portal
   (*Settings → Service Accounts*) with the CLI scopes.
3. Add repository secrets `WIZ_CLIENT_ID` and `WIZ_CLIENT_SECRET`.
4. The workflow in `.github/workflows/wiz.yml` installs `wizcli` and runs:
   - `wizcli iac scan` over Terraform, Kubernetes, and the Compose/Dockerfiles
   - `wizcli dir scan --secrets` over the repository
   - `wizcli docker scan` over both built images
5. If the trial includes the GitHub App for Wiz Code, connect that too and
   compare it against the CLI results.

## What to watch for

- **IaC is where Wiz should win.** 19 findings sit in `infra/` and 8 more in
  `docker-compose.yml`. If Wiz does not comfortably lead the field here, that is
  significant, because this is its home ground.
- **Attack paths.** Wiz's differentiator is connecting findings into a path. Two
  chains in this repository are exactly its shape:
  `J4 → D2` (IMDSv1 enabled + server-side fetch of an arbitrary URL →
  instance credentials → `iam.tf`'s `*:*` policy), and `I1 → I5` (root
  container + mounted Docker socket → host). Does Wiz Code surface either
  *as a path*, or only as unconnected findings? Without a live cloud account it
  may well not — note that honestly either way.
- **Application SAST depth.** Wiz Code's SAST is the newest part of the product.
  Measure the 43 application-layer findings (categories A–G) against what
  CodeQL and Snyk Code found on identical source. This is where the shape
  mismatch will show most clearly.
- **Secrets.** `wizcli dir scan --secrets` should find the eight in category H.

## Cost note

Enterprise contract, sales-led, priced for a cloud estate rather than a
repository. For a self-hosted Compose deployment there is no configuration in
which this is the economical choice. Record it in `SCORECARD.md §4` anyway —
the comparison is worth having on paper for the migration conversation.
---

## What this repository is

A deliberately vulnerable, structurally faithful miniature of **Jivora** — a
Lyme-disease wellness platform handling PHI under HIPAA. It exists to be scanned.

The shape is copied from the real system's architecture document (v1.4), so
findings land on the same surfaces they would in production: a monorepo of three
applications over one PostgreSQL database, two API surfaces with different
authentication, PHI column encryption, an AI assistant with a read-only tool
loop, Stripe webhooks, and a Docker Compose deployment.

```
Jivora-backend/     Node 22 · Express 4 · Sequelize · Socket.IO · @anthropic-ai/sdk
jivora-web/         Next.js App Router · member app · port 4000
jivora-admin/       Next.js App Router · admin panel · port 4002 · Yarn only
infra/terraform/    The AWS deployment the platform would move to
infra/k8s/          The Kubernetes topology for the same
docker-compose.yml  The production stack as actually deployed today
.env.prod           Committed on purpose
```

### Architectural fidelity

| Real Jivora | Here |
| --- | --- |
| Two account stores (`user`, `admin_user`), seven roles | Both, with the same role ids |
| `/api/web/*` and `/api/*` registered separately | Both registrars present |
| PHI column encryption + `*_hash` lookup columns | Present — and subtly wrong |
| Entitlements recomputed live, 402 on paywall | `lib/web/entitlements.js` |
| Admin route manifest as single source of truth | `lib/admin/route-manifest.js` + `route-access.js` |
| 13 read-only assistant tools bound to `user.id` | All 13 — one is not bound |
| Stripe webhook signature mandatory | Claimed in docs, not in code |
| In-process scheduler, no separate worker | `lib/web-protocol/reminder-scheduler.js` |

Where the code contradicts the architecture document, that is the point: the
most interesting findings in this repository are the ones where a stated
security control does not survive contact with the implementation.

## Running it

Don't. There is no seed data, and several dependencies are pinned to versions
with known remote-code-execution advisories. If you want a container for
image-layer scanning:

```sh
docker build -t jivora/backend:sectest Jivora-backend
docker build -t jivora/web:sectest     jivora-web
```

`npm install` inside any of the three apps is enough for dependency scanning and
is safe as long as nothing is executed.

## Scoring a platform

1. Connect the platform. Do not read `FINDINGS.md` first.
2. Let it scan everything it is willing to scan.
3. Export its findings.
4. Now open [`FINDINGS.md`](FINDINGS.md) — the answer key: 57 code and config
   issues across 7 categories, plus 25 dependencies chosen for known advisories.
5. Fill in [`SCORECARD.md`](SCORECARD.md).

Four multi-step attack chains are planted deliberately. Whether a platform
surfaces them *as chains* rather than as unrelated findings is the sharpest
differentiator in this repository.

## Sibling repositories

The same codebase, scanned by a different platform in each:

| Repository | Platform |
| --- | --- |
| [`jivora-sectest-github-advanced-security`](https://github.com/beeva/jivora-sectest-github-advanced-security) | GitHub Advanced Security |
| [`jivora-sectest-aikido`](https://github.com/beeva/jivora-sectest-aikido) | Aikido |
| [`jivora-sectest-snyk`](https://github.com/beeva/jivora-sectest-snyk) | Snyk |
| [`jivora-sectest-veracode`](https://github.com/beeva/jivora-sectest-veracode) | Veracode |
| [`jivora-sectest-wiz`](https://github.com/beeva/jivora-sectest-wiz) | Wiz |

Identical content by design — only this README and the workflows in
`.github/workflows/` differ, so the results are directly comparable.

## Safety

Every credential in this repository is synthetic and authenticates against
nothing. `AKIAIOSFODNN7EXAMPLE` is AWS's own published documentation key. There
is no real PHI here and no real patient data; the clinical vocabulary is
structural, and every value is invented.

GitHub push protection is disabled on this repository on purpose — the committed
`.env.prod` is one of the things being tested for.

Do not copy any pattern from this repository into anything real.
