# RSO-ekipa10/.github

Shared files of the RSO-ekipa10 organization (project Chance).

| Path | What it is |
|---|---|
| `profile/README.md` | Organization profile shown on <https://github.com/RSO-ekipa10> (in Slovenian) |
| `.github/workflows/service-ci-cd.yml` | Shared CI/CD pipeline that every service repository calls |
| `.github/workflows/lint.yml` | Checks the workflows of this repository on pull requests and on `main` |
| `workflow-templates/` | Starter workflow for a new service, shown under *Actions > New workflow* in the organization's repositories |
| `PULL_REQUEST_TEMPLATE.md` | Default pull request checklist for repositories without their own |
| `.github/dependabot.yml`, `.github/CODEOWNERS` | Weekly action updates; reviews by the other team member for pipeline changes |

## The shared pipeline

Each service has a short `.github/workflows/ci-cd.yml` with one job, `pipeline`, that calls `service-ci-cd.yml`. The branch ruleset requires the check `pipeline / test`.

| Event | Jobs | What happens |
|---|---|---|
| Pull request | `test` | workflow lint, `npm ci`, lint, type check, unit tests with coverage (reports kept as the artifact `test-reports`), `helm lint` and a schema check of the chart with the dev and prod values, a check that the prod values hold no local address, image build without push |
| Push to `main` | `test` → `build` → `deploy` | the image is built once and pushed to Docker Hub as `sha-<full commit SHA>`; Helm deploys exactly that image (by digest) to the `prod` namespace on AKS with `--rollback-on-failure --wait`; `helm test` runs the smoke test; if it fails, the release is rolled back to the previous revision |
| Manual run (`workflow_dispatch`) | `deploy` | redeploys the image of an earlier commit of `main` (input `sha`), without building again; the commit must be on `main` and its image on Docker Hub |

The deploy logs in to Azure with OIDC (no stored cloud password), runs in the GitHub environment `production` (only `main` may deploy), and never runs two deploys of a service at once. Before it changes anything, it stops with the exact fix if an earlier deploy was interrupted and left the Helm release stuck (`pending-*`).

Waiting runs: a newer push to a pull request cancels its older run. On `main` (pushes and manual runs) a newer run never cancels a running one, but GitHub keeps only one waiting run: a newer run replaces the waiting one, so the newest commit wins. A replaced merge is not built (it shows as cancelled); its changes ship with the newer commit. A manual run from another branch has its own group and deploys nothing.

### Application settings

An organization secret named after the service (for example `MARKETS_ENV`, lines `KEY=value`) becomes the Kubernetes Secret `<service>-env`; without it the Secret stays as it is.

- Put only secret settings there; non-secret settings belong in `values-prod.yaml`. The deploy masks every single value in the log, so an ordinary short value would be hidden everywhere in the deploy log.
- The deploy writes a hash of the whole secret (never the values) into the Pod annotation `checksum/app-env`, so the Pods restart exactly when the settings change.
- A rollback (automatic, after a failed smoke test, or a manual run with an older `sha`) restores only the image and the chart. The Secret keeps its current settings: to undo a settings change, change the organization secret back and run the workflow manually.

### Settings that depend on the public address (`public-host`, `host-values`)

Values such as the issuer of the tokens (`https://<host>/realms/chance`), Keycloak's own address and the return and checkout URLs depend on the gateway's public host, which is known only at deploy. They are left empty in `values-prod.yaml` (so a deploy without them stops at start instead of answering 401 or redirecting to localhost) and filled in by the pipeline from one organization variable, `PUBLIC_HOST` (a host name only: `chance.example.org`, no scheme, port or path). Each caller lists its own keys:

```yaml
    with:
      public-host: ${{ vars.PUBLIC_HOST }}
      host-values: |
        env.AUTH_ISSUER=https://{host}/realms/chance
```

- Each line `KEY=TEMPLATE` becomes `--set-string KEY=<template with {host} replaced>` for the chart checks with the prod values and for the deploy; the dev values keep their local addresses. Empty lines and lines starting with `#` are skipped.
- The step "Host-dependent values (prod)" checks every line first: the host must be a lowercase name with a dot (no scheme, port or path), a key a value path (`env.AUTH_ISSUER`; `image`, `podAnnotations` and every key inside them are refused, the pipeline sets those itself), and a template may hold only letters, digits, `{host}` and `. _ ~ : / ? # @ % = + -` (so `--set-string` never splits it and the shell never reads it; a URL with `&` cannot be listed). The host values come before the pipeline's own values, so those always win.
- The deploy stops with "set the organization variable PUBLIC_HOST" when a repository lists `host-values` and the variable is missing. The test job (pull requests from Dependabot or forks get no organization variables) then checks every line with the stand-in host `public-host.invalid` and warns. The chart's `values.schema.json` still refuses a key the chart does not have.
- The two copies of the step (test and deploy job) must stay equal: the `lint` workflow of this repository fails when they differ. It compares the whole step (script, `if:`, name, `env`) except the one `env` key that differs on purpose (`REQUIRE_HOST` here, `REQUIRE_VALUES` in "Deploy-time values (prod)") and without YAML comments; the same holds for the two other duplicated steps below.
- The input exists from the commit that added it: a caller that passes it needs that commit of this repository or a later one. Callers on `@main` (all of them until the first release tag) break their `pipeline / test` check with "Invalid input" if they are pushed before this change is on `main`: merge it here first.

### More prod values from organization variables and secrets (`set-values`, `secret-values`)

A chart value that is not host-based but must not be a literal in a public repository comes, one `KEY=VALUE` per line, from an organization variable through the input `set-values` (the value shows in the public run logs) or from an organization secret through the workflow secret `secret-values` (masked, never on a command line). The gateway passes its ACME contact address as a secret:

```yaml
    secrets:
      secret-values: ${{ format('tls.issuer.acme.email={0}', secrets.ACME_EMAIL) }}
```

With `format()` the key stays in the caller even when the secret is missing, so the test job can name it and the deploy can stop on it; an organization secret that held the whole line would give only "No secret-values." when it is missing.

- The step "Deploy-time values (prod)" (in the test and the deploy job, right after "Host-dependent values (prod)"; `set-values` and `secret-values` reach only these two steps, through their own `env`) handles both. A `set-values` line becomes `--set-string KEY=VALUE`. A `secret-values` value is masked for the rest of the job (`::add-mask::`, since GitHub masks only the whole secret; `%`, CR and LF escaped first as `%25`, `%0D`, `%0A`, as `@actions/core` does, so the runner registers the value as it is), written without a final newline to an owner-only file (`0600`, in `$RUNNER_TEMP/secret-values/`) and passed as `--set-file KEY=PATH`: Helm keeps the file content exactly (a final newline would become part of the value), so the shell never parses the value and Helm never splits it at a comma. The step prints the key only; a refused secret line is named by its number.
- What a value is, for both inputs: a line is split at line breaks (a value is one line) and trimmed at both ends, on purpose, so the CR of a secret saved with CRLF line ends or a space left from pasting never becomes part of the value; the value is everything after the first `=` (so it may hold `=`) up to the trimmed end. A value therefore cannot end in a space or tab; in `secret-values` a space right after `=` is kept (`set-values` refuses spaces anyway).
- The order for the chart checks with the prod values and for the deploy: the host values, then `set-values`, then `secret-values`, then the pipeline's own values (`image.repository`, `image.tag`, `image.digest`, `podAnnotations.checksum/app-env`). Helm applies every `--set-file` after every `--set-string`, whatever the order on the command line, so each key may be listed once across `host-values`, `set-values` and `secret-values`, and never together with a key inside it (`env` and `env.AUTH_ISSUER`): such a line is refused. Since none of them may set `image`, `podAnnotations` or a key inside them, the pipeline's own values always win.
- The key checks of `host-values`: a key is a value path; `image`, `podAnnotations` and every key inside them are refused. A `set-values` value may hold only letters, digits and `. _ ~ : / ? # @ % = + -` (no comma, space, quote, `$` or brace).
- An empty value (the variable or secret is missing or not visible to the repository) stops the deploy; the test job warns and leaves the line out, so pull requests from Dependabot or forks (no organization variables or secrets) still pass.
- Organization variables are not masked: a `set-values` value shows in the run logs of a public repository, in the environment of those two steps and in the caller's inputs. A value that must stay out of the public logs goes in `secret-values`.
- `secret-values` keeps a value out of the public logs only. Helm still stores it: in the release history (the release Secrets `sh.helm.release.v1.<service>.v<N>` in the namespace, shown by `helm get values`) and in the rendered objects (the gateway's Issuer carries the address in `spec.acme.email`), readable by everyone who may read Secrets or those objects in the namespace. Credentials (passwords, tokens, keys) therefore stay in `<SERVICE>_ENV` (the Secret `<service>-env`), which Helm never sees.
- The two copies of the step must stay equal; the `lint` workflow diffs them.

### Prod values hold no local address

The step "Prod values hold no local address" (test job, after the chart lint) stops a chart whose prod values still point at the developer's machine. It reads the values exactly as the deploy passes them (the chart's `values.yaml`, `values-prod.yaml`, then the `host-values`, `set-values` and `secret-values` lines, then the image values): a copy of the chart in `$RUNNER_TEMP` gets one more template that prints `.Values` as JSON, and only that document is rendered (`helm template --show-only`). Subchart defaults are part of `.Values`, so they are checked too.

- Every string value that holds a loopback or unspecified address as a whole token fails the job with its key path and value (`env.AUTH_ISSUER=http://localhost:8081/realms/chance`; a multi-line value shows only the lines that hold the address; a value from `secret-values`, or the copy of a `global.*` one in a subchart, is never shown). Caught: `localhost`, `localhost.localdomain`, `ip6-localhost`, `ip6-loopback`, a `.localhost` host, `127.x` in long and short forms (`127.0.0.1`, `127.1`), `0.0.0.0`, `0` as a URL host (`http://0:8080`), `::1`, `[0:0:0:0:0:0:0:1]`, `[::]`. Network ranges and look-alikes pass: `127.0.0.0/8`, `0.0.0.0/0`, `::1/128`, `localhostname`, `localhost.example.org`, `v127.1`, `127.1.2.3.4`, `fe80::1`, `std::vector`, `http://10:80`.
- A value that is right to be local (a sidecar or probe in the same Pod, a listen address) is listed in the chart folder's `prod-local-allow.txt` (`helm/prod-local-allow.txt`), one key path and a reason per line; empty lines and `#` lines are skipped, and a line without a reason fails:

  ```text
  # KEY.PATH                                   REASON
  grafana.sidecar.dashboards.reloadURL          the dashboard sidecar calls Grafana in the same Pod
  prometheus.scrapeConfigs.prometheus.static_configs[0].targets[0]   Prometheus scrapes itself
  ```

  Paths are as the error prints them: keys joined with `.`, list items as `[N]`. A listed path whose value holds no local address any more gives a warning (remove the line).
- The chart must not have a template of its own named `templates/zz-ci-prod-values.yaml` (the check adds that one).

### Third-party charts and CRDs before the chart (`prerequisite-manifests`, `prerequisite-charts`)

A repository whose chart needs other parts in the cluster (the gateway: the Gateway API CRDs, cert-manager and Traefik) lists them, and the pipeline installs them before the repository's own chart on every deploy:

```yaml
    with:
      prerequisite-manifests: |
        https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.3/standard-install.yaml sha256:<64 hex>
      prerequisite-charts: |
        # RELEASE NAMESPACE REPOSITORY CHART VERSION sha256:DIGEST VALUES-FILE
        cert-manager cert-manager oci://quay.io/jetstack/charts cert-manager v1.21.3 sha256:<64 hex> helm/cert-manager-values.yaml
        traefik gateway https://traefik.github.io/charts traefik 42.3.1 sha256:<64 hex> helm/traefik-values.yaml
```

- `prerequisite-manifests`: cluster-wide bundles that ship no chart (CRDs), one `URL sha256:DIGEST` per line, a plain `https://` URL. `prerequisite-charts`: one line per chart, fields separated by spaces: release, namespace, repository (`https://` chart repository or `oci://` registry path), chart, one exact version (no range), the sha256 of the chart archive, a values file in the repository (or `-`). At most 4 of each.
- The step "Prerequisites (fetch and verify)" runs in the test and the deploy job (the two copies must stay equal; the `lint` workflow diffs them): it checks every field, downloads each file (`curl --proto =https`, `helm pull`) and stops when its sha256 is not the pinned one, so a moved release file or a republished chart version never reaches the cluster.
- The test job then validates them with kubeconform (`-strict -ignore-missing-schemas`, like the chart itself; charts rendered with `--include-crds`). CRDs are only parsed: kubeconform's Kubernetes schemas have no schema for `CustomResourceDefinition`.
- The deploy job, after the stuck-release check and before the application Secret:
  1. reads the history of every prerequisite release first: one stuck in `pending-*` stops the deploy before anything changes, with the fix (`helm rollback` to its last good revision, or `helm uninstall` when it was never deployed), as for the repository's own release;
  2. applies the manifests server-side (field manager `ci-cd`, `--force-conflicts`) and waits until their CRDs are Established;
  3. installs the charts in the order listed. Helm installs the CRDs of a chart's `crds/` folder only when they are missing and never upgrades them, so the pipeline applies that folder itself (`helm show crds`, server-side, then the wait) and runs `helm upgrade --install --skip-crds --create-namespace --rollback-on-failure --wait --timeout 5m` (a third-party part may live in its own namespace; the repository's own namespace is still created by hand). A chart upgrade therefore brings its new CRDs too.

  The job's timeout grows from 25 to 55 minutes when prerequisites are listed.
- One owning repository per prerequisite: a CRD bundle or a release that two repositories list is changed back and forth on every deploy of either (`--force-conflicts` takes the fields over, and each `helm upgrade` applies its own values). A part that several charts need is listed by one repository only, and the others rely on it.
- A rollback after a failed smoke test restores only the repository's own release: the prerequisites stay as installed. A manual run (`sha`) installs the prerequisites listed in the newest `ci-cd.yml` with the values files of the commit it deploys.
- The deploy summary gets a row "Prerequisites" when there are any. Dependabot does not see the pins: the caller bumps the version and the digest together (the gateway's README, "Update a prerequisite", has the commands).

### What a service repository needs

- `package.json` with the scripts `lint`, `typecheck` and `test` (the test runner runs once and accepts `--coverage`, for example `vitest run`), and a committed `package-lock.json`
- a `Dockerfile` in the repository root
- a public Docker Hub repository `<DOCKERHUB_USERNAME>/<service>`: the cluster and the manual-run image lookup pull it without credentials (set Docker Hub *Settings > Default privacy* to Public before the first push, or create the repository as public by hand)
- a Helm chart in `helm/` with `values-dev.yaml` and `values-prod.yaml`, and in it:
  - the image values `image.repository`, `image.tag` and `image.digest` (the Deployment uses `repository@digest` when the digest is set, otherwise `repository:tag`)
  - `podAnnotations: {}` in `values.yaml`, rendered into the Pod template of the Deployment (`spec.template.metadata.annotations`, with `with .Values.podAnnotations` and `toYaml . | nindent 8`)
  - `envFrom` with the `secretRef` `<service>-env` and `optional: true` (the Secret exists only when the organization secret is set)
  - a `helm test` hook in `templates/tests/` that calls the service's health endpoint; it must not mount `<service>-env` or print its environment or configuration, because `helm test --logs` writes its output to the public deploy log
- organization settings visible to the repository: the variables `DOCKERHUB_USERNAME`, `AZURE_CLIENT_ID` and, for a repository that passes `host-values`, `PUBLIC_HOST`, and the variables its `set-values` lines read; the secrets `DOCKERHUB_TOKEN`, `AZURE_TENANT_ID` and `AZURE_SUBSCRIPTION_ID` (secrets, so that the public deploy logs mask them), the secrets its `secret-values` lines read (the gateway: `ACME_EMAIL`) and optionally `<SERVICE>_ENV`; and an environment `production` limited to `main`
- no local address in the prod values, or the pod-local ones listed in `helm/prod-local-allow.txt` ("Prod values hold no local address")

### A chart without an image (`build-image: false`)

A repository that is only a Helm chart, such as the message broker (the official NATS chart plus a few templates), has no Node.js code and no image. It calls the same workflow with the input `build-image: false` (default `true`, which is today's behaviour for every other service):

| Step | `build-image: true` (default) | `build-image: false` |
|---|---|---|
| Node.js steps of `test` (setup, `npm ci`, lint, type check, tests with coverage, the `test-reports` artifact) | run | skipped |
| Workflow lint, Helm 4 check, chart lint and schema check, `helm test` hook check | run, the chart gets `image.repository` and `image.tag` | run, the chart gets no image values |
| Image build without push (pull requests), Buildx | run | skipped |
| Job `build` (image built once, pushed to Docker Hub, digest) | on push to `main` | skipped |
| Image digest lookup of a manual run | runs | skipped |
| Job `deploy` | after `build` succeeded | after `test` succeeded (there is no `build` to vouch for it); Helm gets no `image.*` values |

Unchanged: the job ids (`pipeline / test` stays the required check), the events, the permissions (the caller grants `id-token: write` to the `pipeline` job), the concurrency groups, the stuck-release check, the Azure login, `helm upgrade --rollback-on-failure --wait`, `helm test` and the rollback after a failed smoke test. A manual run redeploys the chart of that commit of `main` (input `sha`). The caller passes no `dockerhub-username`, `dockerhub-token`, `image-name` or `node-version` (they are not used), and the repository needs no `package.json`, `package-lock.json` or `Dockerfile`; the chart rules above still apply (`helm/` with `values-dev.yaml` and `values-prod.yaml`, a `helm test` hook), except that no image values are passed, so the chart must not require them, and `podAnnotations` and `envFrom` are needed only when the caller passes `app-env`. The output `image-digest` is empty.

## Changing the pipeline

Changes go through a pull request whose check `actionlint` passes and which the other team member reviews (`.github/CODEOWNERS`). Today this is a team convention: this repository has no branch ruleset, so nothing enforces it. Recommended later: a ruleset on the default branch that requires a pull request, the check `actionlint` and a code-owner review, and a tag ruleset that stops `v*` tags from being moved or deleted.

A reviewed change on `main` gets a release tag (`vX.Y.Z`). The service repositories will pin the shared workflow to the commit SHA of a release tag, so that a change reaches a service only when its Dependabot pull request (or a manual bump) is merged. Until the first release tag exists, `markets` calls `@main`, so every merge here reaches it at its next run. The pin, once the tag exists:

1. `gh api repos/RSO-ekipa10/.github/commits/vX.Y.Z --jq .sha` (gives the commit SHA, also for an annotated tag)
2. in the service's `ci-cd.yml`: `uses: RSO-ekipa10/.github/.github/workflows/service-ci-cd.yml@<40-character SHA> # vX.Y.Z`

A caller that passes an input the pinned commit does not have is refused by GitHub ("Invalid input"): the input `build-image` exists from the commit that added it, so a caller with `build-image: false` points at that commit or a later one. The same holds for `public-host`, `host-values`, `set-values`, `prerequisite-manifests`, `prerequisite-charts` and the secret `secret-values`.

### Tool versions

Dependabot moves the pinned actions (and, in the service repositories, the pinned shared workflow). It does not see the tool versions in the `env` blocks: `HELM_VERSION`, `KUBECTL_VERSION`, `ACTIONLINT_IMAGE` and `KUBECONFORM_IMAGE` in `service-ci-cd.yml`, and `ACTIONLINT_IMAGE` in `lint.yml`. Bump them by hand, monthly (kubectl also when the cluster is upgraded; Helm stays on 4.x, ADR-047). For an image, look up the digest of the new tag, write `<tag>@<digest>`, and let the `actionlint` check of the pull request pass:

```sh
docker buildx imagetools inspect docker.io/rhysd/actionlint:<tag> --format '{{.Manifest.Digest}}'
docker buildx imagetools inspect ghcr.io/yannh/kubeconform:<tag> --format '{{.Manifest.Digest}}'
```

### Organization Actions policy (optional, not enabled)

SHA pinning and the short list of action owners are a convention checked in review, not an enforced setting. An organization owner can enforce both by hand in *Organization settings > Actions > General > Policies*: choose "Allow RSO-ekipa10, and select non-RSO-ekipa10, actions and reusable workflows", allow GitHub's own actions (`actions/*`, `github/*`), allow the patterns `docker/*` and `azure/*` (plus `hashicorp/*` and `terraform-linters/*` for the infrastructure repository), and tick "Require actions to be pinned to a full-length commit SHA". Turn it on only after every service calls the shared workflow at a commit SHA instead of `@main`, and check that the next runs still start.
