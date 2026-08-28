# Setting up ArgoCD on an Ubuntu VM with the Octopus Argo CD Gateway

This guide sets up a single-node ArgoCD instance on an Ubuntu VM (e.g. running in Parallels on Apple Silicon) and connects it to Octopus Deploy via the Octopus Argo CD Gateway. Intended for **non-production / test environments** only.

## Prerequisites

- An Ubuntu VM with at least 2 CPU cores, 4 GB RAM, and **20 GB of disk**. Container images for k3s, ArgoCD, and the gateway add up faster than you'd expect, and Parallels defaults are often smaller than this.
- Outbound internet access from the VM (no inbound access required — the gateway only dials out)
- An Octopus Deploy Cloud instance with permission to manage Infrastructure

## 1. Install k3s

k3s is a lightweight Kubernetes distribution — ideal for a single-node test cluster.

```bash
curl -sfL https://get.k3s.io | sh -
```

Point `kubectl` at the k3s config (k3s writes it to a different location than `kubectl` expects by default):

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

Verify:

```bash
kubectl get nodes
```

You should see the node in `Ready` state. If `kubectl` complains about `localhost:8080`, the kubeconfig step above didn't take — re-check it, or `export KUBECONFIG=/etc/rancher/k3s/k3s.yaml` for the current session.

## 2. Install ArgoCD

**Pin a version.** `stable` moves, so anyone following this guide months from now gets a different ArgoCD than you did — which is the opposite of what a reproducible test environment is for. Check [the releases page](https://github.com/argoproj/argo-cd/releases) for the current version and substitute it below.

```bash
ARGOCD_VERSION=v3.0.6    # <- check for the current release

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/$ARGOCD_VERSION/manifests/install.yaml
```

Wait for the pods to settle:

```bash
kubectl wait --for=condition=available --timeout=300s \
  deployment --all -n argocd
```

**Verify the ApplicationSet controller came with it.** This install route includes it, but confirming now saves a confusing failure later — several scenarios in [SETUP.md](SETUP.md) depend on it, and its absence surfaces as `no matches for kind "ApplicationSet"` much further down the line:

```bash
kubectl get crd applicationsets.argoproj.io
kubectl get pods -n argocd | grep applicationset
```

You want both a CRD and a running pod.

> **Upgrading later** is not just a matter of re-applying `install.yaml` with a new version — CRDs need applying separately with `kubectl apply --server-side -k https://github.com/argoproj/argo-cd/manifests/crds?ref=<version>`. Note the version you installed somewhere; nothing in the cluster makes it obvious later.

## 3. Expose the ArgoCD UI permanently

By default the `argocd-server` service is `ClusterIP` only. For repeat access without re-running `kubectl port-forward` every time, switch it to `NodePort`:

Switch the type and pin a fixed port in one command — a random port gets reassigned if the service is ever recreated:

```bash
kubectl patch svc argocd-server -n argocd -p \
  '{"spec":{"type":"NodePort","ports":[{"name":"https","port":443,"nodePort":30443}]}}'
```

Confirm:

```bash
kubectl get svc argocd-server -n argocd
```

You should see `NodePort` and `443:30443/TCP`.

If ufw is enabled, open the port. The second range covers the demo applications in [SETUP.md](SETUP.md), so you won't have to come back for it:

```bash
sudo ufw allow 30443/tcp
sudo ufw allow 30080:30103/tcp
```

**Finding the VM's IP** — `ip a` is noisy because k3s creates several virtual interfaces (`cni0`, `flannel.1`, `docker0`, `veth*`). Skip the noise with:

```bash
ip route get 1.1.1.1 | grep -oP 'src \K\S+'
```

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Now the UI is reachable at `https://<VM-IP>:30443` (username `admin`) any time, from the VM or any machine on the same network. Expect a self-signed cert warning — fine for a test instance, but **do not expose this on the open internet**.

**Change the password once you're logged in.** You'll need the CLI from step 4:

```bash
argocd account update-password
```

Then delete the bootstrap secret, since it's no longer accurate and shouldn't sit in the cluster:

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

Do this after step 5 if you'd rather set up the Octopus account first — the commands there use your CLI session, not the admin password directly.

## 4. Install the ArgoCD CLI

```bash
uname -m
```

- `aarch64` → download `argocd-linux-arm64` (this is what you'll get on Apple Silicon Parallels VMs by default)
- `x86_64` → download `argocd-linux-amd64`

```bash
curl -sSL -o argocd-linux-<arch> https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-<arch>
sudo install -m 555 argocd-linux-<arch> /usr/local/bin/argocd
rm argocd-linux-<arch>
```

Log in:

```bash
argocd login <VM-IP>:30443 --username admin --password '<password from step 3>' --insecure
```

## 5. Create a scoped ArgoCD user for Octopus

Octopus needs read access to Applications/Clusters/Logs — not a login. Create a dedicated `octopus` account with `apiKey` capability only:

```bash
kubectl patch configmap argocd-cm -n argocd --type merge -p '{"data":{"accounts.octopus":"apiKey","accounts.octopus.enabled":"true"}}'
```

Confirm:

```bash
argocd account list
```

Grant it read (and optionally sync) permissions via RBAC:

```bash
kubectl patch configmap argocd-rbac-cm -n argocd --type merge -p '{"data":{"policy.csv":"p, octopus, applications, get, *, allow\np, octopus, applications, sync, *, allow\np, octopus, clusters, get, *, allow\np, octopus, logs, get, */*, allow\n"}}'
```

> This overwrites the entire `policy.csv` key. If `argocd-rbac-cm` already has custom rules, use `kubectl edit configmap argocd-rbac-cm -n argocd` and append these lines instead of patching. Drop the `sync` line if Octopus should only read state, not trigger syncs.

Generate the token (shown once — copy it now):

```bash
argocd account generate-token --account octopus
```

**This token does not expire**, which is what you want for a long-lived test environment. If you add `--expires-in`, note the date somewhere: an expired token doesn't announce itself, it just makes the gateway stop reporting applications, which looks like an annotation problem rather than an auth one.

Verify the permissions took effect:

```bash
argocd account can-i --auth-token <token> get clusters '*'        # yes
argocd account can-i --auth-token <token> get applications '*'    # yes
argocd account can-i --auth-token <token> get logs '*/*'          # yes
argocd account can-i --auth-token <token> delete applications '*' # no
```

## 6. Install the Octopus Argo CD Gateway

The generated install command uses Helm, which k3s does not ship. Install it first:

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

In Octopus Cloud, go to **Infrastructure → Argo CD Instances → Add Argo CD Instance** and work through the wizard:

- A unique instance name (used for the Kubernetes namespace and Helm release name)
- The environment(s) this gateway will service
- The in-cluster ArgoCD API server URL — default is usually correct (`https://argocd-server.argocd.svc.cluster.local`)
- The token generated in step 5

At the end, Octopus generates a `helm upgrade --install` command. Run it on the VM against the k3s cluster.

If ArgoCD is using a self-signed certificate (the default on a fresh install like this one), add to the generated command:

```
--set gateway.argocd.insecure="true"
```

The gateway makes only outbound connections (to the Octopus REST API on `443`, the Octopus gRPC endpoint on `8443`, and the ArgoCD API server in-cluster) — no inbound firewall rules are needed.

Leave the installation dialog open in Octopus; it waits for a health check to pass. Once green, the gateway is connected and ready to use.

### If the health check never goes green

Check the gateway pod first — it's in a namespace named after the instance you created in the wizard:

```bash
kubectl get pods -n <instance-name>
kubectl logs -n <instance-name> -l app.kubernetes.io/name=argocd-gateway --tail=100
```

Three causes account for most failures:

- **Self-signed certificate.** A fresh ArgoCD install uses one, and without `--set gateway.argocd.insecure="true"` the gateway can't complete the TLS handshake to the ArgoCD API. The logs mention certificate validation.
- **Bad or insufficient token.** Re-run the `can-i` checks from step 5. A token that reads clusters but not applications produces a gateway that starts cleanly and then reports nothing.
- **Outbound 8443 blocked.** The gateway needs the Octopus gRPC endpoint as well as HTTPS on 443. Corporate networks and VPNs block 8443 more often than 443, and the symptom is a gateway that looks healthy in the cluster while Octopus never sees it.

The pod restarting in a loop points at the first two; a running pod with nothing appearing in Octopus points at the third.

## 7. Next steps

With the gateway healthy, the remaining work is wiring up scoping annotations on your ArgoCD Applications so Octopus knows which projects and environments each one maps to.

If you're setting up the demo environment, continue with **[SETUP.md](SETUP.md)** in this repository — it picks up exactly here, covering the annotations, the Octopus projects, and six worked scenarios. For the annotation reference on its own, see the [Octopus annotations documentation](https://octopus.com/docs/argo-cd/annotations).

## References

- [Octopus Argo CD Instances — Overview](https://octopus.com/docs/argo-cd/instances)
- [Argo CD Authentication for Octopus](https://octopus.com/docs/argo-cd/instances/argo-user)
- [Automated Installation (Helm/Terraform/Argo Application)](https://github.com/OctopusDeploy/docs/blob/main/src/pages/docs/argo-cd/instances/automated-installation.md)
