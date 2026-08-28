# Argo CD + Octopus Deploy demo environment — setup guide

This repository stands up a set of Argo CD applications for demonstrating the
Octopus Deploy Argo CD integration. It covers six scenarios of increasing
complexity, all sharing one small nginx workload so the moving parts stay
visible.

The first three are Argo CD only — no Octopus involvement at all. The last
three each need **their own Octopus project**, so three projects in total.

Read [Part 1](#part-1--prerequisites) through [Part 4](#part-4--octopus-setup)
in order the first time. After that, the
[troubleshooting section](#troubleshooting) is the part worth bookmarking — it
lists every problem we actually hit, with the symptom you'll see rather than
the cause you'd have to guess at.

---

## What you get

| # | Scenario | What it demonstrates | Argo CD apps | Octopus project |
| --- | --- | --- | --- | --- |
| 1 | **Single application** | Plain GitOps. Edit a file, commit, sync. | `argocd-demo` | none (optional — see below) |
| 2 | **App of apps** | Argo CD managing its own Application definitions. | `root` | none |
| 3 | **ApplicationSet** | Apps generated from Git directories; fan-out changes. | `demo-development`, `demo-test`, `demo-production` | none |
| 4 | **Octopus image tags** | Release promotion by image tag across environments. | (uses the scenario 3 apps) | **project A** |
| 5 | **Octopus manifests** | Octopus generating whole manifests from templates. | `octopus-development`, `octopus-test`, `octopus-production` | **project B** |
| 6 | **Helm** | Helm charts, per-environment values, Helm-specific Octopus config. | `helm-development`, `helm-test`, `helm-production` | **project C** |

Ten Argo CD applications in total, from one manual `kubectl apply`.

### How Octopus and Argo CD fit together

Worth getting straight before Part 4, because the two products divide the work
in a way that isn't obvious from either UI on its own.

**Argo CD is the only thing that talks to the cluster.** Octopus never applies
a manifest. What it does is commit to Git — a new image tag, or a whole
rendered manifest — and then ask Argo CD to sync. Argo CD does the rest, exactly
as it would for a change you typed yourself.

So a deployment is really four steps:

```
Octopus release  →  commit to Git  →  Argo CD syncs  →  pods roll
```

**The gateway is how Octopus reaches Argo CD.** Octopus Cloud has no inbound
route into your cluster, so a small agent — the Octopus Argo CD Gateway — runs
alongside Argo CD and dials out to Octopus. That's what registers your instance
under Infrastructure → Argo CD Instances and lets Octopus enumerate
applications. Installing it is covered in
[`argocd-octopus-gateway-setup.md`](argocd-octopus-gateway-setup.md).

**The annotations are the join between the two halves.** Octopus has no idea
what your directory structure looks like. When a deployment runs, it scans the
Argo CD applications the gateway reports and looks for two annotations on each
one:

```yaml
argo.octopus.com/project: your-project-slug
argo.octopus.com/environment: development
```

An application is in scope for a deployment when **both** match — the project
being deployed, and the environment being deployed to. That is the entire
matching rule, and it explains most of the behaviour that surprises people:

- **One project, three environments, three applications.** Deploy to Test and
  Octopus acts on the one application annotated `test`, ignoring the others.
  That's what makes promotion work.
- **Two applications sharing a project and environment both get updated** in a
  single deployment. Useful on purpose, confusing by accident — it's why each
  scenario here gets its own project.
- **An unannotated application is invisible to Octopus.** It isn't an error;
  Octopus simply has nothing to act on.

**Where you see this in the Octopus UI.** Two places worth opening early:

| Where | What it shows |
| --- | --- |
| **Argo CD Applications view** (under the Argo CD instance) | Every application Octopus can see, with the project and environment it resolved from the annotations, and any outstanding configuration — missing Git credentials, for instance. This is where you confirm the annotations landed. |
| **Deployment task log** | Which applications the step matched, what it committed, and what it skipped. A step that matched nothing still succeeds, so the log is where you find that out. |

The Applications view is the one to check first whenever something doesn't
behave. If an application isn't listed there, no amount of deployment-process
configuration will make it work — the annotation is wrong, or Octopus hasn't
refreshed yet.

**Two things to know about that view.** It caches, so a newly annotated
application can take a minute to appear; a failed verification that then
succeeds unchanged is usually just the cache catching up. And it lists
applications from the whole Argo CD instance, not just yours — so scenarios
1–3 will show up unannotated and unclaimed, which is correct.

### How much Octopus do I need?

**Three environments, one lifecycle, and three projects** — all sharing the
same three environments and the same lifecycle.

| You need | How many | Notes |
| --- | --- | --- |
| Environments | 3 | Development, Test, Production. Shared by all projects. |
| Lifecycle | 1 | Three phases, one environment each. Shared by all projects. |
| Docker Hub feed | 1 | Shared. Used by scenarios 4 and 6. |
| Git credentials | 1 | Shared. Needs **write** access — Octopus commits. |
| Projects | 3 | One each for scenarios 4, 5, and 6. |

**Scenarios 1–3 involve no Octopus configuration whatsoever.** They're worth
standing up first regardless — they're how you verify Argo CD, the repository
URL, and the ApplicationSet controller all work before adding a second product
to the picture.

Scenario 1 can optionally be attached to project A as a single-environment
worked example of the image-tag step — see
[Single application](#1-single-application). It stays outside Octopus unless
you choose to annotate it.

**Why scenarios 4–6 can't share one project** follows from the matching rule
above: a shared project slug would make one deployment update every scenario at
once. The Helm scenario also requires a step setting the others must not have.
Full explanation in
[One Octopus project per scenario](#one-octopus-project-per-scenario).

### Repository layout

```
argocd-octopus-gateway-setup.md   build the environment from a bare VM (start here
                                  if you have no cluster yet)

bootstrap/
  root-app.yaml              applied by hand, once — bootstraps everything else

argocd/                      synced by root; all Application/ApplicationSet CRs
  application.yaml           the single application
  applicationset.yaml        the tenant ApplicationSet
  applicationset-octopus.yaml  the Octopus-manifests ApplicationSet
  applicationset-helm.yaml   the Helm ApplicationSet

single/                      scenario 1 — kustomize, hand-managed
appset/                      scenario 3 — kustomize base + per-env overlays
  base/
  tenants/development|test|production/
octopus-managed/             scenario 5 — written by Octopus, do not hand-edit
  development|test|production/
templates/                   scenario 5 — input templates for Octopus
helm/demo-web/               scenario 6 — Helm chart + per-env values files
```

### Port map

Every service is a NodePort so pages stay reachable across rollouts. Bookmark
these once.

| Port | Application | Namespace |
| --- | --- | --- |
| 30080 | single | `argocd-demo` |
| 30081 / 30082 / 30083 | ApplicationSet tenants | `demo-development` / `-test` / `-production` |
| 30091 / 30092 / 30093 | Octopus-managed | `octopus-development` / `-test` / `-production` |
| 30101 / 30102 / 30103 | Helm | `helm-development` / `-test` / `-production` |

If you followed the gateway guide, the Argo CD UI itself is on `30443`. Nothing
here collides with it.

---

## Part 1 — Prerequisites

**Starting from nothing?** [`argocd-octopus-gateway-setup.md`](argocd-octopus-gateway-setup.md)
in this repository walks through building the whole environment from a bare
Ubuntu VM: k3s, Argo CD, a scoped Argo CD service account for Octopus, and the
Octopus Argo CD Gateway that connects the two. It ends exactly where this guide
begins — with a healthy gateway and no annotations wired up yet.

Follow that first if you don't already have Argo CD registered in Octopus. The
rest of Part 1 is for people who already have a cluster and just need to check
it's suitable.

### Cluster

Any Kubernetes cluster where NodePorts are reachable from wherever you'll open
a browser. k3s, k3d (with `-p` port mappings), minikube, and real nodes all
work as-is. **kind and Docker Desktop need port mappings declared at cluster
creation time** — if you're using either, sort that out before going further
or none of the demo pages will load.

### Argo CD — including the ApplicationSet controller

Install Argo CD however you normally would, then **verify the ApplicationSet
controller is present**. Not every installation route includes it, and its
absence produces a confusing error much later:

```bash
kubectl get crd applicationsets.argoproj.io
kubectl get pods -n argocd | grep applicationset
```

You need both a CRD and a running pod. If either is missing, see
[ApplicationSet CRD missing](#applicationset-crd-missing) before continuing —
three of the six scenarios depend on it.

The gateway guide's install route — the upstream `install.yaml` — includes the
controller, so if you followed it this check should pass. Helm chart and
`namespace-install.yaml` routes are the ones that commonly don't.

### Octopus Deploy

An Octopus instance with your cluster registered under **Infrastructure → Argo
CD Instances**, connected by the Octopus Argo CD Gateway. See
[`argocd-octopus-gateway-setup.md`](argocd-octopus-gateway-setup.md) if that
isn't in place yet — Octopus can't see any application in your cluster until it
is, annotations or not.

You'll also need permission to create projects, environments, feeds, and Git
credentials.

---

## Part 2 — Repository setup

### 1. Get your own copy

Fork this repository, or create a new one and upload the contents. **Put the
contents at the repository root** — `single/`, `appset/`, `argocd/` and the
rest should be top-level directories, not nested inside a wrapper folder.

Then clone it locally. Everything from here on assumes you're working inside
that clone, and you'll want it anyway for the editing demos.

```bash
git clone https://github.com/YOUR-ORG/YOUR-REPO.git
cd YOUR-REPO
```

Where you clone it matters slightly:

- **Linux, macOS, or a Linux VM.** Anywhere in your home directory is fine.
- **WSL.** Clone inside the WSL filesystem (`~/`), not under `/mnt/c/`.
  Cross-filesystem access is slow and Git's file-mode handling gets confused.
- **Windows.** Anywhere works. If `git` isn't installed, get it from
  [git-scm.com](https://git-scm.com/download/win) — the installer includes Git
  Bash, which will also run the bash commands in this guide if you'd rather not
  translate them.

**One Windows-specific setting worth checking.** Git for Windows converts line
endings to CRLF on checkout by default. That's harmless for the YAML in this
repository, but if you later add shell scripts they'll fail in containers with
a `bad interpreter` error. To keep checkouts as-is:

```powershell
git config --global core.autocrlf input
```

### 2. Replace the repository URL

The placeholder `YOUR-ORG/argocd-demo-app` appears in several files, and
**each ApplicationSet contains two occurrences** — one on the generator, one
on the template. Missing the second one is easy and fails in a confusing way:
the ApplicationSet generates apps successfully, then every generated app
errors on repository access.

In every command below, the **first** URL is the placeholder being replaced and
the **second** is your repository. Reversing them is easy to do and puts the
placeholder back into files you'd already fixed.

**Linux or WSL (GNU sed):**

```bash
sed -i 's|https://github.com/YOUR-ORG/argocd-demo-app.git|https://github.com/YOUR-ORG/YOUR-REPO.git|g' \
  argocd/*.yaml bootstrap/root-app.yaml
grep -rn "YOUR-ORG" . || echo "all clear"
```

**macOS (BSD sed):**

macOS ships BSD sed, where `-i` requires a backup-suffix argument. Pass an
empty string for no backup — note the space between `-i` and `''`:

```bash
sed -i '' 's|https://github.com/YOUR-ORG/argocd-demo-app.git|https://github.com/YOUR-ORG/YOUR-REPO.git|g' \
  argocd/*.yaml bootstrap/root-app.yaml
grep -rn "YOUR-ORG" . || echo "all clear"
```

Without the `''`, BSD sed reads the next argument as the backup suffix and then
tries to parse your first filename as the script, giving:

```
sed: 1: "argocd/application.yaml
": command a expects \ followed by text
```

If you'd rather not think about the difference, `perl` behaves the same on both
platforms:

```bash
perl -pi -e 's|\Qhttps://github.com/YOUR-ORG/argocd-demo-app.git\E|https://github.com/YOUR-ORG/YOUR-REPO.git|g' \
  argocd/*.yaml bootstrap/root-app.yaml
```

**Windows PowerShell:**

```powershell
$old = 'https://github.com/YOUR-ORG/argocd-demo-app.git'
$new = 'https://github.com/YOUR-ORG/YOUR-REPO.git'

Get-ChildItem argocd\*.yaml, bootstrap\root-app.yaml | ForEach-Object {
    $text = Get-Content $_.FullName -Raw
    # -replace uses regex, so escape the pattern; the URL contains dots and slashes.
    $text -replace [regex]::Escape($old), $new |
        Set-Content $_.FullName -NoNewline
}

# Verify nothing was missed
$hits = Get-ChildItem -Recurse -Include *.yaml | Select-String 'YOUR-ORG'
if ($hits) { $hits } else { 'all clear' }
```

`Set-Content` writes UTF-8 in PowerShell 6+. On Windows PowerShell 5.1 it
defaults to ANSI, which is fine for these files (they're plain ASCII) but add
`-Encoding utf8` if you edit them further.

Check `targetRevision` matches your default branch while you're in there.

### 3. Private repository? Add credentials to Argo CD now

```bash
argocd repo add https://github.com/YOUR-ORG/YOUR-REPO.git \
  --username YOUR-USER --password YOUR-PAT
```

Skip this and applications sit in `Unknown` state with a repository access
error rather than anything obviously auth-related.

---

## Part 3 — Bootstrap with app of apps

This is the only manual `kubectl apply` in the whole setup. Everything else
follows from it.

### 1. Push your URL changes first

Argo CD reads from GitHub, not from your working copy. If the repository URL
edits from Part 2 are still uncommitted, the bootstrap will appear to succeed
and then every application will fail on repository access.

```bash
git status                      # should show your edited YAML files
git add -A
git commit -m "Point manifests at my repository"
git push
```

### 2. Apply the root application

Run this from the root of your clone, on a machine where `kubectl` targets the
right cluster:

```bash
kubectl apply -f bootstrap/root-app.yaml
```

The path is relative, so `kubectl` must be run from the repository root. In
PowerShell the same command works with either slash direction; use
`bootstrap\root-app.yaml` if tab-completion gives you backslashes.

Check you're pointed at the intended cluster before applying:

```bash
kubectl config current-context
```

**If `kubectl` and your clone are on different machines** — a common setup when
Argo CD runs on a Linux VM and you edit on a Windows laptop — you have two
options. Either clone the repository on the VM as well and apply from there, or
apply straight from GitHub without a local file:

```bash
kubectl apply -f https://raw.githubusercontent.com/YOUR-ORG/YOUR-REPO/main/bootstrap/root-app.yaml
```

That raw URL only works for public repositories; see
[`kubectl apply -f <raw GitHub URL>` returns 404](#kubectl-apply--f-raw-github-url-returns-404)
if it doesn't.

The `root` application syncs the `argocd/` directory, so all four
Application and ApplicationSet definitions get created for you. Adding a new
scenario later is a matter of committing a file to `argocd/` — no further
manual applies.

### Why `bootstrap/` is separate from `argocd/`

If the root application's own manifest lived in the directory it syncs, it
would manage itself, and a bad edit could leave you unpicking a
self-referential loop by hand. Keeping it one directory over makes it a plain
bootstrap artifact: apply once, forget.

### What to expect

```bash
kubectl get applications -n argocd
```

You should see `root` plus nine child applications. **They will all show
`Missing` health.** This is correct — automated sync is deliberately switched
off so you can demonstrate the `OutOfSync` state rather than having everything
silently reconcile. Sync them when you're ready:

```bash
argocd app sync argocd-demo
argocd app sync demo-development demo-test demo-production
argocd app sync helm-development helm-test helm-production
```

The `octopus-*` applications will show empty or placeholder content until
their first Octopus deployment writes manifests into them.

### About automated sync

`root` runs with `selfHeal: true` so that edits to `argocd/` reach the cluster
without intervention — this is what makes "edit a file on GitHub, watch Argo CD
update its own configuration" work.

Two consequences worth knowing:

- **Editing an Application through the Argo CD UI no longer sticks.** Changes
  get reverted within seconds. Annotations in particular must be committed to
  Git, not added through the UI.
- **Pruning is off by default.** Deleting a file from `argocd/` leaves an
  orphaned Application behind. Turn `prune: true` on once you've decided
  whether you want deletions to cascade — and if you do, add
  `finalizers: [resources-finalizer.argocd.argoproj.io]` to the child
  applications, or you'll get workloads running that no Application claims.

---

## Part 4 — Octopus setup

Only scenarios 4, 5, and 6 use Octopus. If you've just finished Part 3, you
already have a working environment for scenarios 1–3 and can stop here until
you want the Octopus half.

Everything in this part is shared across the three Octopus projects except the
projects themselves — see
[How much Octopus do I need?](#how-much-octopus-do-i-need).

### Environments and lifecycle

Create three environments: **Development**, **Test**, **Production**.

Then create a lifecycle (Library → Lifecycles) with three phases, one
environment each, in that order. Without it you get the default lifecycle,
which works but gives you no promotion gating — and gating is most of the
story you're demonstrating.

### Feeds and credentials

- **Docker Hub feed** (Library → External Feeds → Docker Container Registry,
  `https://index.docker.io`). Anonymous works for public nginx; authenticating
  avoids rate limits.
- **Git credentials** (Library → Git Credentials) with **write** access to the
  repository. Octopus commits to it — read-only access is not enough.

### Scoping annotations

The matching rule is covered in
[How Octopus and Argo CD fit together](#how-octopus-and-argo-cd-fit-together);
this is where to put the annotations in practice.

Values are **slugs, not display names** — check the slug field on the project
and environment pages. A display name of "Development" is usually the slug
`development`, but confirm rather than assume.

```yaml
argo.octopus.com/project: your-project-slug
argo.octopus.com/environment: development
```

**Where they go depends on how the application was created:**

| Application created by | Annotations go on |
| --- | --- |
| A hand-written Application CR | `metadata.annotations` on the Application |
| An ApplicationSet | `spec.template.metadata.annotations` on the ApplicationSet |

Putting them on an ApplicationSet's own `metadata.annotations` is valid YAML,
applies cleanly, and does nothing — see
[Annotations on the wrong level](#annotations-on-the-wrong-level).

In this repository the ApplicationSets template the environment from the
element name, so all three environments are handled by one block:

```yaml
  template:
    metadata:
      name: 'demo-{{.env}}'
      annotations:
        argo.octopus.com/project: your-project-slug
        argo.octopus.com/environment: '{{.env}}'
```

You only need to set the project slug. Set it once per ApplicationSet — each
scenario should be its own Octopus project (see below).

**Verify before moving on.** Commit, let `root` sync, then check the annotation
reached the generated applications rather than just the ApplicationSet:

```bash
kubectl get application demo-development -n argocd \
  -o jsonpath='{.metadata.annotations}'
```

Then open the **Argo CD Applications view** in Octopus. The applications should
be listed with their resolved project and environment. If they aren't, give it
a minute for the cache and refresh before assuming the annotation is wrong.

### One Octopus project per scenario

Three projects, one each for scenarios 4, 5, and 6. Scenarios 1–3 need none.

| Project | Scenario | Step | Annotate |
| --- | --- | --- | --- |
| A | 4 — image tags | Update Argo CD Application Image Tags | `argocd/applicationset.yaml` |
| B | 5 — manifests | Update Argo CD Application Manifests | `argocd/applicationset-octopus.yaml` |
| C | 6 — Helm | Update Argo CD Application Image Tags | `argocd/applicationset-helm.yaml` |

All three share the same environments, lifecycle, feed, and Git credentials.
Only the project differs — set each ApplicationSet's
`argo.octopus.com/project` annotation to that project's slug.

Two reasons they can't be one project:

1. **The Helm scenario needs configuration the others must not have.** The
   image-tag step's Helm image value field is required for Helm sources and
   must be empty for Kustomize and directory sources. Same step, incompatible
   settings — they can't share a deployment process.
2. **The step acts on every application matching the project and environment
   slugs.** Share a slug across scenarios and one deployment updates all of
   them at once. That's a legitimate pattern to demonstrate deliberately, but a
   confusing accident to stumble into.

---

## The scenarios

Every scenario's page is reachable at `http://localhost:<port>` from the
machine running the cluster — see the [port map](#port-map) for the full list.
Each section below repeats its own ports. If a page won't load, check the
NodePort caveat in [Part 1](#cluster).

### 1. Single application

`http://localhost:30080`

Plain kustomize, manually synced. The page is served from a ConfigMap
generated from `single/files/index.html`.

**Demo loop:** edit the release string and colour at the top of the HTML file,
commit, refresh in the Argo CD UI, sync, reload the browser tab.

Other changes worth showing:

| Edit | What Argo CD shows |
| --- | --- |
| `replicas` in `deployment.yaml` | pod count changes in the resource tree |
| `newTag` in `kustomization.yaml` | rolling update, old ReplicaSet scaled to 0 |
| remove `service.yaml` from `kustomization.yaml` | resource marked for pruning |
| `kubectl scale` the deployment by hand | drift — `OutOfSync` with no commit |

**Why the ConfigMap has a hash suffix.** The kustomization uses
`configMapGenerator`, which appends a content hash to the ConfigMap name.
Changing the HTML changes the name, which changes the Deployment's volume
reference, which forces a rolling update. Without it, Argo CD reports `Synced`
while the running pods keep serving the old page — the single most confusing
failure in a GitOps demo. Do not set `disableNameSuffixHash: true` unless you
want to demonstrate exactly that.

**The image tag lives in `kustomization.yaml`, not `deployment.yaml`.** The
Octopus image-tag step reads the kustomization's `images` transformer, so
keeping the tag there means this app can be driven from Octopus as well as by
hand — useful as the simplest possible worked example before the
multi-environment scenarios:

```yaml
images:
  - name: nginx
    newTag: 1.27.4
```

Rendering it locally is a good way to see the transformer applied without
involving Argo CD at all:

```bash
kustomize build single | grep image:
```

To drive it from Octopus, uncomment the annotation block in
`argocd/application.yaml` and point it at your image-tag project. **Be
deliberate about the environment slug:** the step acts on every application
matching the project and environment, so annotating this app as `development`
means a single deployment updates both it and `demo-development`. That's a
legitimate demo of one release fanning out across applications — just not one
you want to discover by accident.

### 2. App of apps

No page of its own — watch this one in the Argo CD UI.

**What's different about this scenario.** Every other scenario manages
*workloads* — Deployments, Services, ConfigMaps. This one manages *Argo CD's
own configuration*. The `root` application's source path is `argocd/`, the
directory holding your Application and ApplicationSet definitions, so Argo CD
treats its own control-plane objects as just more manifests to reconcile.

That's what removes the manual `kubectl apply` from everything else. Adding a
scenario later means committing a file to `argocd/` — `root` notices and
creates the Application for you.

**The demo.** Add an annotation to `argocd/application.yaml` on GitHub. The file
ships with no `annotations:` block on the Application, so you're adding one
rather than changing an existing value. Any harmless key will do:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd-demo
  namespace: argocd
  annotations:
    demo.example.com/touched-by: app-of-apps    # <- add these two lines
spec:
  ...
```

Commit it. Within a minute `root` picks it up and adds that annotation to the
`argocd-demo` Application object in the cluster. Nobody ran a command.

Repeat the demo by changing the value — `app-of-apps-2` — and the same thing
happens for an edit rather than an addition.

Note the commented-out `argo.octopus.com/*` block already in that file is for a
different purpose (scenario 4); leave it alone for this demo, since uncommenting
it brings the app into scope for Octopus deployments.

**What you should and shouldn't see.** This is the part worth being explicit
about, because a successful run looks like almost nothing happening:

- **The `root` application** goes `OutOfSync`, then `Synced` again.
- **The `argocd-demo` Application object** gains the new annotation. Confirm it
  in the UI under Details → Summary, or:

  ```bash
  kubectl get application argocd-demo -n argocd \
    -o jsonpath='{.metadata.annotations}'
  ```

- **The `argocd-demo` application itself stays `Synced` and `Healthy`**, and
  **the running pods do not restart**. The page at
  `http://localhost:30080` is unchanged.

That last point is correct behaviour, not a failed deployment. You added
metadata to the Application resource, not anything in `single/` — so the
manifests Argo CD renders for the workload are byte-for-byte identical and
there is nothing to roll. If pods *had* restarted, that would be the surprising
outcome.

To see `root` drive a visible change instead, edit something in `spec` rather
than `metadata` — `revisionHistoryLimit`, or the destination namespace. Those
alter the Application's actual configuration and you'll see Argo CD act on it.

**One consequence to keep in mind.** `root` runs with `selfHeal: true`, so
edits made through the Argo CD UI to anything under `argocd/` get reverted
within seconds. See [About automated sync](#about-automated-sync).

### 3. ApplicationSet

`http://localhost:30081` / `:30082` / `:30083` — development, test, production

A Git directory generator globs `appset/tenants/*`, so **each directory is an
application**. The directories are named for the Octopus environments, which
lets the ApplicationSet template the environment annotation from the directory
name.

Three demos, different in kind:

1. **Change one tenant.** Edit `appset/tenants/test/files/index.html`. Only
   `demo-test` goes `OutOfSync`.
2. **Change the base.** Edit the `resources` block in
   `appset/base/deployment.yaml` — bump the memory limit from `64Mi` to
   `128Mi`, say. All three tenants go `OutOfSync` at once and every pod rolls.
   This fan-out is the argument for ApplicationSets.
3. **Add or remove a tenant.** Copy a tenant directory, commit, and a fourth
   application appears on its own with its own namespace. Delete it and it goes
   away. Nothing in scenario 1 can do this.

**Pick your base edit carefully — three fields are overridden per tenant.**
Each overlay's `kustomization.yaml` transforms the base, so a change to any of
these in `appset/base/deployment.yaml` renders away to nothing and the demo
appears not to work:

| Field | Overridden by | Change it here instead |
| --- | --- | --- |
| `replicas` | `replicas:` transformer | the tenant's `kustomization.yaml` |
| `image` tag | `images:` transformer | the tenant's `kustomization.yaml` |
| `nodePort` | `patches:` block | the tenant's `kustomization.yaml` |

Anything else in the base propagates cleanly. `resources`, the readiness probe,
and `containerPort` are all safe choices for a fan-out demo; `resources` is the
clearest because it forces a visible rollout in all three namespaces at once.

Confirm the base actually won on whichever field you picked:

```bash
kustomize build appset/tenants/development | grep -A6 resources:
```

**NodePorts live in the overlays, not the base.** Node ports are cluster-wide,
so three tenants sharing a base with a fixed `nodePort` would collide — one
binds, the others fail to sync. The base sets `type: NodePort`; each overlay
patches in its own number. Same reasoning applies to the replica count and
image tag, which are per-tenant values by design.

### 4. Octopus image tag promotion

`http://localhost:30081` / `:30082` / `:30083` — the scenario 3 pages

Uses the ApplicationSet applications from scenario 3.

**Octopus setup: the first of your three projects.** Create it on the
three-phase lifecycle from Part 4, with a single *Update Argo CD Application
Image Tags* step and an nginx package reference from your Docker Hub feed. Put
its slug in the `argo.octopus.com/project` annotation in
`argocd/applicationset.yaml`.

- **No deployment targets needed.** The step runs on the Octopus Server and
  finds its work by annotation. If it's asking for a target role, something is
  misconfigured.
- **No environment scoping needed on the step.** It resolves the application
  from the environment the deployment is running in. One step covers all three.

**Use numeric image tags.** `1.27-alpine` isn't valid semver, so Octopus can't
order versions reliably. Each tenant's `kustomization.yaml` carries an images
transformer:

```yaml
images:
  - name: nginx
    newTag: 1.27.4
```

**The tag must live inside each tenant's own path.** Each application's source
path is `appset/tenants/<env>`, so a tag defined in `appset/base/` is outside
the path Octopus is given and won't be updated.

**Demo:** create a release with a new nginx version, deploy to Development,
then promote the same release through Test and Production. Watch the three
pages update one environment at a time.

Add a **Manual Intervention** step scoped to Production only, placed before the
image step. It's skipped in Development and Test, so promotion stays quick
while Production stays governed — the clearest single illustration of what
Octopus adds on top of Argo CD.

### 5. Octopus manifest generation

`http://localhost:30091` / `:30092` / `:30093` — development, test, production
(blank until the first Octopus deployment writes manifests)

Octopus renders `templates/app.yaml` with Octopus variables substituted and
commits the result into `octopus-managed/<env>/`.

**Octopus setup: a second project**, separate from scenario 4's. It uses a
different step, so there's nothing to share. The environments, lifecycle, and
Git credentials are the same ones you already created; put this project's slug
in the `argo.octopus.com/project` annotation in
`argocd/applicationset-octopus.yaml`.

Add an *Update Argo CD Application Manifests*
step. Template source is this repository, branch `main`, folder `templates`.
Enable **purge** so removed resources are removed from the target directory.

Project variables, scoped by environment:

| Variable | development | test | production |
| --- | --- | --- | --- |
| `Replicas` | 1 | 2 | 3 |
| `NodePort` | 30091 | 30092 | 30093 |
| `BandColour` | `#4c9a8f` | `#c2734a` | `#7a6bbd` |
| `ImageTag` | 1.27.4 | 1.27.4 | 1.27.4 |

**Sensitive variables fail the deployment.** Keep everything a plain project
variable.

**This ApplicationSet uses a list generator, not a directory generator.**
Octopus creates the contents of the output directories, so a directory
generator would find nothing before the first deployment ran.

**Nothing forces a pod restart here.** There's no kustomize hash on a plain
directory source, so the template stamps `#{Octopus.Release.Number}` into a pod
annotation. New release, new pod spec, guaranteed rollout. If you write your
own templates, keep that trick or your ConfigMap changes won't take effect.

**Placeholder files.** Each output directory needs a file so Argo CD can
resolve the path before Octopus first writes to it. Use `README.md` — Argo CD's
directory source only reads `.yaml`, `.yml`, and `.json`, so markdown holds the
directory open without being parsed as a manifest. Purge removes it on first
deployment, which is fine.

**Release versioning.** This project has no package reference driving it, so
leave it on Octopus's default incrementing number. Consider starting it at
`1.0.0` so it's visually distinguishable from your other projects on screen —
the release number is rendered in 5rem type on the page.

### 6. Helm

`http://localhost:30101` / `:30102` / `:30103` — development, test, production

A chart at `helm/demo-web/` with a values file per environment. This is the
scenario where the Octopus integration genuinely behaves differently.

**Octopus setup: a third project, built the same way as scenario 4.** This
isn't a variation on the project you already made — it's a new one, because the
step needs a setting scenario 4's project must not have. Everything else is
identical, and the environments, lifecycle, feed, and Git credentials are
shared, so there's nothing new to create in Library.

Set it up exactly as in [scenario 4](#4-octopus-image-tag-promotion):

- A new project on the same three-phase lifecycle.
- One *Update Argo CD Application Image Tags* step, no deployment targets, no
  environment scoping on the step.
- An nginx package reference from the same Docker Hub feed.
- The project's slug in the `argo.octopus.com/project` annotation in
  `argocd/applicationset-helm.yaml` — not the slug you used for scenario 4.

**Then the one setting that differs:** on the step, open the nginx package
reference and populate the **Helm image value** field:

```
image.tag
```

This field is required for Helm sources and must be left empty for Kustomize
and directory sources — which is exactly why this can't share scenario 4's
project. Without it the deployment succeeds while doing nothing, logging a
warning about a missing image replace path annotation.

Release versioning, promotion, and the optional Production manual-intervention
step all work as they do in scenario 4.

**`values.yaml` has no `tag` key on purpose.** The image path is defined per
chart, not per values file, so Octopus updates `image.tag` wherever it finds it
across that chart's values files — including the shared defaults. That would
leave `values.yaml` defaulting to whatever was last deployed to Development,
which any environment that stopped pinning its own tag would silently inherit.
Removing the key from the base closes it; the template falls back to
`.Chart.AppVersion`:

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

**`checksum/config` on the pod template** is the Helm equivalent of the
kustomize ConfigMap hash — when the rendered ConfigMap changes, the checksum
changes, the pod spec changes, and the Deployment rolls.

**`releaseName` is set per environment.** Without it Argo CD uses the
Application name as the Helm release name, which surprises people who then
can't find it.

Worth being able to explain in customer conversations: **Argo CD renders charts
with `helm template`, it doesn't install them.** `helm list` shows nothing, and
there is no Helm release history to roll back to. Rollback comes from Git and
from Octopus. This catches out a lot of teams moving from the Helm CLI.

---

## Troubleshooting

### ApplicationSet CRD missing

**Symptom:** `no matches for kind "ApplicationSet" in version "argoproj.io/v1alpha1"`,
or via the root application, the more cryptic
`one or more synchronization tasks are not valid: ApplicationSet.argoproj.io "" not found`.

**Cause:** the installation route used didn't include the ApplicationSet
controller. Helm doesn't install or update CRDs by default; `namespace-install.yaml`
omits them entirely.

**Fix:** find your Argo CD version, then install matching CRDs.

```bash
kubectl get deploy argocd-repo-server -n argocd \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

kubectl apply --server-side --force-conflicts \
  -k https://github.com/argoproj/argo-cd/manifests/crds?ref=vX.Y.Z

kubectl rollout restart deployment argocd-applicationset-controller -n argocd
```

`--server-side` is not optional — Argo CD's CRDs exceed the annotation size
limit that client-side apply uses.

If the controller *deployment* doesn't exist either, re-apply the full install
manifest for your version. Note it will reassert ownership of resources like
`argocd-cm`, so diff first if your install is customised.

**After fixing:** the CRD alone is enough for `kubectl apply` to succeed. An
ApplicationSet will sit there generating nothing if the controller pod isn't
running. Confirm `1/1 Running` before concluding a manifest is wrong.

### Annotations on the wrong level

**Symptom:** Octopus doesn't see the applications. The ApplicationSet looks
correctly annotated.

**Cause:** annotations placed on the ApplicationSet's own
`metadata.annotations` rather than `spec.template.metadata.annotations`. Both
are valid and both apply cleanly. Octopus only reads Application objects.

**Check what actually reached the generated apps:**

```bash
kubectl get application demo-development -n argocd \
  -o jsonpath='{.metadata.annotations}' | jq
```

### Octopus doesn't pick up new annotations immediately

Octopus caches its view of Argo CD applications. A manual verification can fail
and then succeed a minute later with no change on your side. Refresh the Argo CD
Applications view in Octopus before assuming something is misconfigured.

### Helm: deployment succeeds but nothing changes in Git

**Symptom:** task log warns that a Helm source `is missing an annotation for
the image replace path. It will not be updated.`

**Cause:** no Helm image value configured on the package reference. Octopus
falls back to looking for an annotation and finds none.

**Fix:** set the Helm image value field to `image.tag` on the package
reference, then **create a new release** — the setting is snapshotted at
release creation, so existing releases won't pick it up.

**Note the annotation route takes a different format entirely.** The step field
takes a plain YAML path (`image.tag`). The
`argo.octopus.com/image-replace-paths` annotation takes a Helm-template string
building a *fully qualified* image name, because Octopus needs the registry to
avoid updating images from other registries:

```yaml
argo.octopus.com/image-replace-paths: "docker.io/{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Putting `image.tag` in the annotation silently fails to match. Also: if any
package in a step uses annotations, all of them must.

### Page shows the old version after a successful sync

Work down the chain:

```bash
# 1. What image are the pods actually running?
kubectl get pods -n helm-development \
  -o jsonpath='{.items[*].spec.containers[*].image}{"\n"}'

# 2. What did Octopus commit?
git log --oneline -3 && git show --stat HEAD

# 3. Is it just the browser?
curl -s localhost:30101 | grep -o '1\.27\.[0-9]'
```

In PowerShell, the first two work as written; the third needs different tools,
since `curl` is an alias for `Invoke-WebRequest` and doesn't take `-s`:

```powershell
(Invoke-WebRequest localhost:30101).Content | Select-String -Pattern '1\.27\.\d'
```

If pods are on the old image, Git wasn't updated where Argo CD reads. If pods
are on the new image and the fetched page shows the new version, it's browser
cache.

If pods are on the new image but the page is stale, the ConfigMap didn't
re-render — check the rollout trigger (kustomize hash, Helm `checksum/config`,
or the release-number pod annotation, depending on the scenario).

### Edited the ApplicationSet base and nothing happened

**Symptom:** you changed a value in `appset/base/deployment.yaml`, committed,
and the tenants stayed `Synced` — or synced without any visible change.

**Cause:** the field you edited is overridden by a transformer in each tenant's
`kustomization.yaml`. `replicas`, the image tag, and `nodePort` are all
per-tenant values; a base edit to any of them renders away.

**Fix:** change the value in the tenant's `kustomization.yaml` instead, or pick
a field the overlays don't touch — `resources` is the easiest. Check what
actually renders:

```bash
kustomize build appset/tenants/development
```

### Applications show `Missing` health

They've been generated but never synced. Automated sync is off by design. Sync
them.

### `kubectl port-forward` keeps dying

It pins to a single pod even when you target a Service, so it drops every time
a rollout replaces that pod — which is every time you demo anything. Use the
NodePorts instead; that's why they're configured.

### Editing an Application in the Argo CD UI doesn't stick

`root` runs `selfHeal: true`. Commit to Git instead. This is correct behaviour
and is itself a good drift-correction demo.

### Argo CD reports `Synced` but the page is unchanged

Almost always a ConfigMap without a content hash. See the rollout-trigger notes
in scenarios 1, 5, and 6.

### `kubectl apply -f <raw GitHub URL>` returns 404

- Private repositories return 404 rather than 403 for unauthenticated raw
  requests.
- Case matters in the path.
- The file may simply not be pushed yet.

Test with a file you know exists — `curl -I .../README.md`, or in PowerShell
`Invoke-WebRequest .../README.md -Method Head`. Cloning the
repository and applying from a local path avoids the whole question — and
you'll want it cloned locally for editing demos anyway.

### Sync fails with a node port conflict

Node ports are cluster-wide. Two applications can't share one, regardless of
namespace. Check the [port map](#port-map).

---

## Notes on the images used

The `nginx` image runs as root. That's fine for a demo cluster, but if your
namespaces have restrictive Pod Security admission, swap to
`nginxinc/nginx-unprivileged` and change `containerPort` to `8080`.
