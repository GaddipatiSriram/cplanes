# EKS bootstrap & ArgoCD registration — AWS-recommended patterns

> Teaching doc. Reference for "how does the big-cloud version of what we
> built actually work, and how do enterprises run *many* of these at once?"
>
> Sibling to [`cluster/README.md`](./README.md) (homelab overview) and
> [`k8s-bstrp/docs/README.md`](./k8s-bstrp/docs/README.md) (our kubeadm
> playbook on oVirt). Read that first if you want our side of the contrast.
>
> All command snippets are copy-paste real — no invented flags — against
> `eksctl ≥0.195`, `aws-cli v2`, `terraform ≥1.7`, `argocd ≥2.13`, and
> EKS ≥1.30. Where AWS has recently deprecated an approach, both the
> legacy and current commands are shown so you can recognize both in the
> wild.

---

## Why read this

You've read `k8s-bstrp/docs/README.md`. You know how we use `kubeadm init`
on Rocky VMs, drop a Flannel manifest, and register the cluster with ArgoCD
via an out-of-band SA+token. That works for a homelab.

In a real AWS shop, **nobody runs kubeadm**. The control plane is managed.
The node fleet is usually autoscaled. Identity federates to IAM rather than
static kubeconfigs. Addons are first-class API resources. And the *norm*,
not the exception, is that clusters are created and destroyed on demand —
one per PR, one per team, one per region, one per prod-blue / prod-green
cutover. This doc walks through the AWS-recommended way each of those
pieces works, and how ArgoCD fits in as a central controller that watches
a fleet of clusters it didn't create.

We'll map everything back to our Pattern A/B/C/D selectors at the end
(see [`argo-applications/README.md`](../layers/argo-applications/README.md) for those).

---

## 1 — The EKS cluster lifecycle, 7 stages

```
        ┌────────────────────────────────────────────────┐
        │                  AWS account                   │
        │                                                │
  (1) Control plane       (2) Node capacity              │
  ┌──────────────┐        ┌────────────────────────────┐ │
  │  eks-ctrl    │◄──────►│  Managed NG │ Fargate │ K  │ │
  │  (AWS-mgd)   │        └────────────────────────────┘ │
  └──────┬───────┘                                       │
         │                                                │
  (3) CA + kubeconfig        (4) Identity                 │
  aws eks get-token ◄─── STS ◄─── IAM ── aws-auth /       │
         │                              Access Entries,   │
         │                              IRSA, Pod Identity│
  (5) EKS addons   (6) Platform add-ons   (7) GitOps      │
  vpc-cni          AWS LB Controller      ArgoCD registers│
  coredns          external-dns           this cluster +  │
  kube-proxy       cert-manager           ApplicationSets │
  ebs-csi          external-secrets       sync everything │
  pod-id-agent     metrics-server         from here on    │
  └───────────────────────────────────────────────────────┘
```

### (1) Control-plane provisioning

AWS provisions and operates the control plane (3 API servers, etcd, the
cloud-controller-manager). You do **not** get SSH onto control-plane
nodes, and you cannot run DaemonSets on them. You do get:

- A **cluster endpoint** URL (e.g. `https://ABC123.gr7.us-east-1.eks.amazonaws.com`)
  with a choice of access mode:
  - **Public** — anyone on the internet can reach the API (still authn/z-gated).
  - **Private** — only resources inside the VPC (+ peered VPCs / TGW).
  - **Public + private** — default for many setups; the in-VPC clients
    route privately, outside clients route publicly.
- A **cluster CA certificate** returned by `DescribeCluster` — kubelets
  and clients use it to verify the API server.
- **Envelope encryption** of etcd Secrets with a KMS CMK (AWS strongly
  recommends enabling this at create time; it's irreversible).
- **Control-plane logs** to CloudWatch: `api`, `audit`, `authenticator`,
  `controllerManager`, `scheduler`. All off by default; most compliance
  regimes require `audit` + `authenticator` on.
- **VPC and subnets** are prerequisites — you give EKS ≥2 subnets in ≥2
  AZs for control-plane ENIs. Tag them `kubernetes.io/role/elb=1`
  (public) or `kubernetes.io/role/internal-elb=1` (private) so the AWS
  Load Balancer Controller can place LBs correctly later.

Create the bare cluster with `eksctl` (good for learning, bad for prod):

```bash
eksctl create cluster \
  --name demo --region us-east-1 --version 1.31 \
  --vpc-private-subnets subnet-aaa,subnet-bbb,subnet-ccc \
  --vpc-public-subnets  subnet-ddd,subnet-eee,subnet-fff \
  --with-oidc \
  --without-nodegroup
```

`--with-oidc` is important — it creates the OIDC provider in IAM that
IRSA (stage 4) needs. `--without-nodegroup` gives you just the control
plane; you'll add capacity next.

In prod, you'd use the `terraform-aws-modules/eks/aws` module instead.
State-in-S3 / lock-in-DynamoDB gives you immutable, auditable infra.
Crossplane does the same job *from* an already-existing cluster — covered
in §3.

### (2) Node capacity — three flavors, pick your trade-off

EKS decouples control plane from compute. You opt into one (or more) of:

| Flavor | Who owns the ASG? | Node lifecycle | When it fits |
|---|---|---|---|
| **Managed node group** | AWS | AMI upgrades triggered by you, done by AWS with rolling drain | Mainstream default. Worker nodes feel "managed" without giving up DaemonSet access. |
| **Self-managed node group** | You | Fully your ASG + user-data | You need custom AMIs (e.g. FIPS, bespoke CNI) or a hand-tuned launch template. |
| **Fargate profile** | None (no nodes) | Per-pod microVM | Low-traffic control-plane addons (CoreDNS!), pipelines, per-tenant workloads with strong isolation. |
| **Karpenter** (autoscaler) | A controller in-cluster | Just-in-time provisioning per-pod | **AWS-recommended default** for elastic / spiky workloads. Replaces cluster-autoscaler for most new clusters. |

Managed node group via `eksctl`:

```bash
eksctl create nodegroup \
  --cluster demo --region us-east-1 \
  --name general-on-demand \
  --node-type m6i.large --nodes 3 --nodes-min 3 --nodes-max 10 \
  --managed \
  --asg-access --full-ecr-access --alb-ingress-access
```

Karpenter works differently: you install it as a Helm chart, then create
`NodePool` and `EC2NodeClass` CRDs describing which instance types it
may spin up. Karpenter watches unschedulable pods and provisions nodes
that fit, often landing cheaper than any static ASG.

```yaml
# Karpenter NodePool — a capacity budget, not a node count
apiVersion: karpenter.sh/v1
kind: NodePool
metadata: { name: default }
spec:
  template:
    spec:
      requirements:
        - { key: karpenter.sh/capacity-type, operator: In, values: [on-demand, spot] }
        - { key: karpenter.k8s.aws/instance-family, operator: In, values: [m6i, m7i, c7i] }
      nodeClassRef: { group: karpenter.k8s.aws, kind: EC2NodeClass, name: default }
  limits: { cpu: "1000" }
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
```

AWS's current guidance is: Managed NG for predictable baseline, Karpenter
for burst, Fargate for CoreDNS and similar system pods if you want zero
node management for them.

Homelab analogue: we don't have an autoscaler. `bootstrap-k8s.yml` creates
a fixed set of nodes per cluster (3 control plane + N workers); there is
no cost pressure to scale down. That's fine at homelab scale but not what
production feels like.

### (3) Cluster CA & kubeconfig

The `aws eks update-kubeconfig` command writes a kubeconfig using the
**exec auth plugin**. The kubeconfig doesn't contain a static bearer
token — it contains an exec stanza:

```yaml
users:
- name: arn:aws:eks:us-east-1:1234:cluster/demo
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: aws
      args: [ "eks", "get-token", "--cluster-name", "demo" ]
```

Every `kubectl` call shells out to `aws eks get-token`, which signs a
**pre-signed STS GetCallerIdentity URL** with your current AWS creds.
That URL is sent as a bearer token to the API server. The EKS control
plane's authentication webhook (`aws-iam-authenticator`) re-signs and
validates the URL against STS, then maps the caller's IAM principal to
a Kubernetes identity via either the aws-auth ConfigMap (legacy) or
Access Entries (current).

Why this matters: your Kubernetes credentials are *your IAM credentials*.
Rotate an IAM key → kubeconfig keeps working (or breaks) by policy, not
by file edit.

```bash
aws eks update-kubeconfig --name demo --region us-east-1 --alias demo
kubectl get nodes   # signs STS on every call
```

### (4) IAM ↔ Kubernetes identity

Three mechanisms, each solving a different slice:

**(4a) aws-auth ConfigMap — legacy, still dominant in older clusters.**
A ConfigMap in `kube-system` mapping IAM roles/users to Kubernetes
groups. The authenticator webhook reads it on every request.

```yaml
# kube-system/aws-auth ConfigMap
apiVersion: v1
kind: ConfigMap
metadata: { name: aws-auth, namespace: kube-system }
data:
  mapRoles: |
    - rolearn: arn:aws:iam::1234:role/demo-nodes-role
      username: system:node:{{EC2PrivateDNSName}}
      groups: [ system:bootstrappers, system:nodes ]
    - rolearn: arn:aws:iam::1234:role/Admin
      username: admin
      groups: [ system:masters ]
  mapUsers: |
    - userarn: arn:aws:iam::1234:user/jane
      username: jane
      groups: [ view ]
```

Problems: it's a single ConfigMap — all mutations race. A typo locks
everyone out. Permissions are "pick a group string"; there's no Describe
API, no audit trail beyond CloudTrail seeing ConfigMap edits.

**(4b) EKS Access Entries — current AWS recommendation (GA Nov 2023).**
First-class AWS API resources:

```bash
# Grant an IAM role cluster-admin via Access Policy
aws eks create-access-entry \
  --cluster-name demo --region us-east-1 \
  --principal-arn arn:aws:iam::1234:role/PlatformAdmin \
  --type STANDARD

aws eks associate-access-policy \
  --cluster-name demo --region us-east-1 \
  --principal-arn arn:aws:iam::1234:role/PlatformAdmin \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

Scoped: `AmazonEKSViewPolicy`, `AmazonEKSEditPolicy`, `AmazonEKSAdminPolicy`,
`AmazonEKSClusterAdminPolicy`, `AmazonEKSAdminViewPolicy`. Scopes support
`type=namespace,namespaces=foo,bar` for per-namespace grants.

Set authentication mode on cluster create:

- `CONFIG_MAP` — aws-auth only (legacy)
- `API` — Access Entries only (recommended for new clusters)
- `API_AND_CONFIG_MAP` — both (migration mode; Access Entries take
  precedence on conflict)

**(4c) IRSA — IAM Roles for Service Accounts.** This is how your pods
get AWS permissions without any long-lived key on the node. A
ServiceAccount gets annotated with an IAM role ARN:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::1234:role/external-dns-r53
```

The pod ends up with two projected env vars (`AWS_ROLE_ARN`,
`AWS_WEB_IDENTITY_TOKEN_FILE`). The AWS SDK sees them and calls STS
`AssumeRoleWithWebIdentity`, trading the projected ServiceAccount JWT
(signed by the cluster's OIDC provider, which AWS registered as an
`iam:OIDCProvider` on cluster create) for short-lived IAM credentials.
This is how the entire AWS controller zoo — AWS LB Controller,
external-dns, external-secrets, cluster-autoscaler, Karpenter — gets its
AWS permissions.

**(4d) EKS Pod Identity — simpler IRSA replacement (Nov 2023).** An
EKS addon (`eks-pod-identity-agent`) runs as a DaemonSet and handles
credential brokering locally, so pods talk to a link-local IMDS-like
endpoint instead of doing the OIDC dance. You associate an IAM role to
a ServiceAccount with `aws eks create-pod-identity-association`, no
annotation edits required.

```bash
aws eks create-pod-identity-association \
  --cluster-name demo --region us-east-1 \
  --namespace kube-system --service-account external-dns \
  --role-arn arn:aws:iam::1234:role/external-dns-r53
```

AWS is nudging new clusters toward Pod Identity over IRSA — fewer
cross-account/cross-region edge cases, no per-cluster OIDC provider
registration.

Homelab: we have none of this. Our workloads don't talk to cloud APIs,
and our ServiceAccount tokens are plain in-cluster JWTs with RBAC.
When we get to "Vault issues cloud-like short-lived creds to pods",
that *is* the IRSA pattern, ported.

### (5) EKS addons

Historically you'd helm-install VPC CNI, CoreDNS, kube-proxy yourself.
Today these are **EKS addons** — AWS-managed Helm charts with version
catalogs and supported upgrade paths. Addons show up as both AWS API
resources and in-cluster DaemonSets / Deployments.

Common addons to install alongside or just after cluster create:

```bash
for a in vpc-cni coredns kube-proxy aws-ebs-csi-driver eks-pod-identity-agent; do
  aws eks create-addon --cluster-name demo --region us-east-1 --addon-name $a
done
```

Also common: `amazon-cloudwatch-observability`, `aws-efs-csi-driver`,
`adot` (AWS Distro for OpenTelemetry), `guardduty-agent`.

Terraform equivalent:

```hcl
resource "aws_eks_addon" "vpc_cni" {
  cluster_name             = module.eks.cluster_name
  addon_name               = "vpc-cni"
  addon_version            = "v1.19.2-eksbuild.1"
  service_account_role_arn = aws_iam_role.vpc_cni.arn
  resolve_conflicts_on_update = "OVERWRITE"
}
```

Key design pattern: **`service_account_role_arn` wires the addon to an
IRSA-backed IAM role.** That's how the addon gets exactly the AWS perms
it needs, no more.

Homelab analogue: our addons are ApplicationSets. Where AWS says "EKS
addons are first-class cluster resources", we say "Pattern A
ApplicationSets with selector `tier In [mgmt, dev], role NotIn [storage]`
give us the same 'install everywhere' effect." EKS addons catalog vs
Pattern A AppSets — different mechanisms, same goal.

### (6) Platform services on top

Once the cluster has addons, you install the services that make it useful:

| Component | Role | AWS idiom |
|---|---|---|
| **AWS Load Balancer Controller** | creates ALBs/NLBs from Ingress + Service type=LoadBalancer | IRSA role, Helm chart, or `terraform-aws-load-balancer-controller` |
| **external-dns** | pushes DNS records to Route53 | IRSA role w/ `route53:ChangeResourceRecordSets` |
| **cert-manager** | TLS automation; ACME or AWS Private CA issuer | same chart as homelab, different Issuer |
| **ExternalSecrets Operator** | pulls AWS Secrets Manager / SSM Parameter Store into k8s Secrets | IRSA role w/ `secretsmanager:GetSecretValue` |
| **metrics-server** | HPA plumbing | official chart |
| **Karpenter** / cluster-autoscaler | horizontal scale | IRSA role w/ EC2 RunInstances, TerminateInstances scoped by tag |

In the AWS-recommended **EKS Blueprints for Terraform** flow, these are
all wired up as modules that emit outputs consumed by each other (e.g.
`aws-load-balancer-controller` module depends on the `vpc-cni` addon
being installed). Blueprints is the closest analogue to our
`forge/` + `argo-applications/sre/forge/` stack — a pre-wired "platform in a box."

### (7) GitOps handoff

You've got a cluster with addons and a platform layer. From this point,
**stop installing anything manually.** Register the cluster with a
central ArgoCD; an ApplicationSet fleet takes over the rest.

This is §5 — the "registration patterns" are important enough to get
their own section.

---

## 2 — AWS-recommended provisioning tools

Contrast, not a bake-off. Pick per scenario:

| Tool | When it's the AWS-recommended choice |
|---|---|
| **`eksctl`** | Day-1 / learning / a single cluster owned by a single team. Official AWS CLI wrapper. `eksctl create cluster -f cluster.yaml` is fine for demos and test envs. |
| **Terraform + `terraform-aws-modules/eks/aws`** | Enterprise default. Immutable infra, state as source of truth, blast radius per workspace, code review per cluster. Most production EKS lives here. |
| **EKS Blueprints for Terraform** | Opinionated preset: Terraform + addon catalog + ArgoCD + platform services, AWS-maintained. Closest to a "stack-as-code" starting point. When you want Day-2 ready without wiring 15 modules yourself. |
| **Crossplane** | K8s-native IaC. A mgmt-plane cluster defines a `Composition` that creates EKS clusters from a `Claim` CR. Enterprise use case: "factory" cluster that creates N dynamic EKS clusters on demand, each with its own platform layer. |
| **ACK (AWS Controllers for Kubernetes)** | Narrower than Crossplane — AWS-maintained controllers that expose AWS services as CRDs. Good when you want CRDs for AWS resources without Crossplane's composition model. |
| **Cluster API + CAPA** | Multi-cloud shops standardizing on Cluster API treat EKS as one `AWSManagedControlPlane` flavor. Uniform cluster lifecycle across AWS/GCP/Azure/on-prem. |

The enterprise trend is *away* from imperative `eksctl` and toward either
Terraform+Blueprints (IaC shop) or Crossplane/CAPA (platform-team shop).

---

## 3 — Enterprise use cases for dynamic EKS clusters

Creating one cluster is easy. The interesting problems start when you
run a **fleet** that changes daily.

### 3a — Preview environments per PR

A PR opens → CI creates a short-lived EKS cluster → deploys the app →
runs integration tests → tears the cluster down when the PR merges.
Variants: "preview namespace per PR on a shared cluster" is cheaper;
"preview cluster per PR" is what's needed when you're testing
cluster-scoped things (CRDs, operators, network policies).

Typical implementation:

- **Crossplane** composition `XCluster` creates EKS + VPC + addons.
- GitHub Actions creates an `XClusterClaim` with `spec.parameters.prNumber`.
- ArgoCD `ApplicationSet` with a **cluster generator** +
  `matchLabels: { env: preview }` fans out the app manifests as soon
  as the new cluster secret exists.
- PR close → claim deleted → Crossplane tears down → ArgoCD drops
  Applications (cluster Secret gone, cluster generator stops matching).

### 3b — Per-team / per-tenant clusters

Hard-multi-tenancy shops (regulated, or teams with conflicting CRD
needs) give each team a dedicated cluster. Central ArgoCD + Vault + OPA
Gatekeeper / Kyverno enforce posture uniformly. Tenant onboarding is:
new entry in a Terraform module → pipeline runs → cluster exists →
tenant's app Applications sync.

### 3c — Multi-region active/active

Fleet of identical clusters, one per region, all subscribed to the same
Git source. `ApplicationSet` cluster generator picks up a `region`
label from the cluster Secret; per-region parameterization goes through
the selector.

```yaml
generators:
- clusters:
    selector: { matchLabels: { tier: prod } }
template:
  metadata:
    name: '{{name}}-app'
  spec:
    source:
      path: 'envs/{{metadata.labels.region}}'
```

### 3d — Blue/green cluster cutover

Spin up `prod-green` alongside `prod-blue`. ArgoCD syncs all apps to
`prod-green`. Flip Route53 / ALB weights. Once `prod-green` is serving
cleanly, delete `prod-blue`. This pattern is why AWS pushes so hard on
"nothing has ClickOps state on the cluster" — if your cluster has any
state that only exists on `prod-blue`, you can't cutover.

### 3e — Reference architectures worth reading

- **EKS Blueprints "Cluster Factory"** — Terraform + Crossplane pattern,
  one mgmt-plane cluster provisions N workload clusters.
- **Upbound's Universal Control Plane** — managed Crossplane-as-SaaS.
- **Backstage + Crossplane** — Backstage scaffolder templates create
  `Claim`s; developers self-serve clusters.
- **Karpenter for everything** — per-cluster cost posture without
  per-tenant capacity planning.

---

## 4 — Cluster → ArgoCD registration patterns

The actual knot: **once the cluster exists, how does ArgoCD learn about
it?** ArgoCD treats clusters as labeled Secrets in its own namespace. So
"register a cluster" = "produce a Secret that looks like this":

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: prod-green
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
    tier: prod
    region: us-east-1
    env: production
type: Opaque
stringData:
  name: prod-green
  server: https://ABC123.gr7.us-east-1.eks.amazonaws.com
  config: |
    {
      "bearerToken": "...",
      "tlsClientConfig": {"insecure": false, "caData": "..."}
    }
```

Four patterns get you there, in rough order of enterprise maturity:

### Pattern X — push-from-the-cluster (what our homelab does)

After `kubeadm init` completes, a localhost Ansible play:

1. Creates `kube-system/argocd-manager` ServiceAccount + ClusterRoleBinding
   to `cluster-admin` on the **new** cluster.
2. Creates a long-lived token Secret (`type: kubernetes.io/service-account-token`).
3. Reads the token + the cluster's CA bundle.
4. Constructs the `type: cluster` Secret above and `kubectl apply`s it
   to the **central ArgoCD** namespace.

Pros: no cloud APIs needed, dead simple, works for on-prem and cloud.
Cons: long-lived SA tokens are a security smell. Teardown has no
automation — you have to `kubectl delete secret <cluster>` when the
cluster goes away, or ArgoCD retries the dead endpoint forever.

Our implementation: `cluster/k8s-bstrp/bootstrap-k8s.yml` Phase 8 —
see [`cluster/README.md`](./README.md) for the label convention.

### Pattern Y — pull-from-ArgoCD with `argocd cluster add`

The `argocd` CLI reads your local kubeconfig, connects to the target
cluster, creates the SA + token + RBAC on that cluster, then registers
it:

```bash
argocd cluster add arn:aws:eks:us-east-1:1234:cluster/demo \
  --name demo --label tier=prod --label region=us-east-1
```

Pros: one command. Cons: interactive, requires your kubeconfig to be
set up, awkward for CI-driven automation at scale. Fine for operator
onboarding one or two clusters by hand.

### Pattern Z — Cluster-create emits the Secret (Crossplane / CAPA)

This is the enterprise end-state. In a Crossplane `Composition` or CAPA
`Cluster` resource, the *act of creating the cluster* includes a step
that renders the ArgoCD cluster Secret and applies it to the central
ArgoCD namespace. Argo ApplicationSets with a cluster generator pick it
up automatically.

Sketch (Crossplane):

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata: { name: xeks-with-argo }
spec:
  resources:
    - name: cluster
      base:
        apiVersion: eks.aws.upbound.io/v1beta1
        kind: Cluster
        # ...
    - name: nodegroup
      base: { apiVersion: eks.aws.upbound.io/v1beta1, kind: NodeGroup }
    - name: argo-cluster-secret
      base:
        apiVersion: kubernetes.crossplane.io/v1alpha2
        kind: Object
        spec:
          providerConfigRef: { name: argo-hub }
          forProvider:
            manifest:
              apiVersion: v1
              kind: Secret
              # rendered with bearerToken, caData from cluster outputs
```

Teardown mirror: deleting the `Claim` deletes both the cluster AND the
ArgoCD Secret, so ArgoCD no longer sees a cluster it can't reach.

### Pattern W — AWS-specific, IRSA-based cluster auth (no SA tokens)

The ArgoCD controller on the hub cluster has an IRSA role that can
`sts:AssumeRole` into each workload account. The cluster Secret stores
the role ARN and trust, not a token:

```yaml
stringData:
  name: prod-green
  server: https://ABC123.gr7.us-east-1.eks.amazonaws.com
  config: |
    {
      "awsAuthConfig": {
        "clusterName": "prod-green",
        "roleARN": "arn:aws:iam::1234:role/ArgoCDManager"
      },
      "tlsClientConfig": { "caData": "..." }
    }
```

On every API call, the ArgoCD server runs `aws eks get-token` with the
AssumeRole-acquired creds. No long-lived tokens anywhere. This is the
right answer for prod EKS fleets. Setup cost: an IAM role per workload
account with a trust policy accepting the hub's IRSA role as principal,
and an Access Entry / aws-auth mapping for that role on the workload
cluster.

### Teardown is half the problem

A common mistake: your create pipeline is slick, but your destroy
pipeline forgets to remove the cluster Secret. ArgoCD starts logging
"connection refused" on a dead endpoint, the controller hotloops, the
UI shows a zombie cluster. Patterns Z and W handle this automatically;
Patterns X and Y require an explicit delete step.

Our own Phase 8 **does not delete on teardown today** — something we
should fix when we do the mgmt-observability rebuild.

---

## 5 — EKS vs homelab — side-by-side

| Concern | EKS (AWS-recommended) | Homelab (this repo) |
|---|---|---|
| Control plane | `aws eks create-cluster` (managed; AWS runs 3 apiservers + etcd) | `kubeadm init` on Rocky VM, Flannel |
| Node join | Managed NG / Karpenter with signed bootstrap scripts | `kubeadm join` from Phase 6 with shared token |
| Cluster CA | AWS-issued, rotated automatically | `kubeadm` self-signed, 10-year default |
| Client auth | IAM → STS → bearer via exec plugin | Static admin kubeconfig + SA tokens |
| RBAC ↔ IAM mapping | `aws-auth` ConfigMap or Access Entries | Direct RoleBindings on k8s groups |
| SA → cloud IAM | IRSA (OIDC trust) or Pod Identity (agent) | N/A — no cloud IAM; Vault is the analogue |
| Addons | `aws_eks_addon` catalog (vpc-cni, coredns, ebs-csi, …) | Pattern A ApplicationSets (cert-manager, ingress-nginx, external-secrets, rook-ceph-external, trivy-operator) |
| Load balancer | AWS LB Controller → ALB/NLB | MetalLB VIP + ingress-nginx |
| DNS | external-dns → Route53 | CoreDNS on `mgmt-core`, MetalLB VIP `10.10.1.200` |
| TLS certs | cert-manager + ACM PCA / Let's Encrypt | cert-manager + (target) Vault PKI Issuer |
| Secrets | ESO → Secrets Manager / SSM | ESO → Vault |
| Observability | CloudWatch Container Insights + AMP + AMG | Prometheus + Grafana on `mgmt-observability` |
| Cluster registration | Crossplane/CAPA emits Secret, or IRSA trust | `bootstrap-k8s.yml` Phase 8 pushes SA-token Secret |
| Teardown de-registration | Composition cleanup deletes Secret | Manual `kubectl delete secret` (gap) |

Same shape, different substrates. The interesting rows are the ones we
have nothing for today:

- **No IRSA analogue** — our pods don't talk to cloud APIs. When Vault
  matures (short-lived creds, PKI, Kubernetes auth method), we'll
  effectively have the "Vault-backed pod identity" pattern, which is
  what IRSA+Pod Identity are solving on the AWS side.
- **No addon catalog** — Pattern A AppSets are our answer but there's
  no version-catalog / supported-version pinning the way EKS enforces.
  Renovate + ApplicationSet chart version bumps is the nearest thing.
- **No dynamic cluster factory** — Phase 8 is manual-trigger today. A
  "create-cluster wrapper that also writes the argocd cluster Secret
  and deletes it on destroy" would be the missing link.

---

## 6 — What this implies for our roadmap

Things worth porting from the AWS idiom into our `cluster/k8s-bstrp/`
and `ovirt-setup/playbooks/` over the next few sessions:

1. **De-registration in Phase 8** — `bootstrap-k8s.yml` currently only
   creates the ArgoCD cluster Secret. A sister playbook (or
   `cleanup_vms.yml` extension) should delete it before the VMs go
   away. This is the "Pattern X teardown gap" called out in §4.
2. **Single wrapper: `full-deploy.yml`** — imports VM creation + disk
   grow + kubeadm + ArgoCD register, taking one per-cluster vars file.
   EKS's `eksctl create cluster -f config.yaml` is the inspiration: one
   manifest → one cluster, end to end.
3. **Label schema discipline** — our current `cluster_labels` (tier,
   role, env) maps 1:1 to the AWS `aws_eks_cluster` tags that drive
   EKS Blueprints' selector patterns. Keep them tight; don't let
   per-cluster ad-hoc labels creep in. This is what makes Pattern A/B/C/D
   selectors scale.
4. **Explicit addon catalog** — today `argo-applications/sre/forge/` hosts one
   AppSet per addon. Consider a single "platform-addons" AppSet with
   a Git file generator over a manifest of pinned versions, so upgrades
   are "edit one file → ArgoCD rolls the fleet." That's the EKS addon
   catalog model in homelab form.
5. **Vault as the IRSA analogue** — when Vault's Kubernetes auth method
   is wired up and short-lived creds are flowing, we get the IRSA /
   Pod Identity shape. Plan the observability rollout with that in mind
   so Grafana/Prometheus don't end up with static admin passwords.

Out of scope for this doc: actually doing the mgmt-observability
rebuild + disk-prep refactor. That's its own plan.

---

## See also

- [`cluster/README.md`](./README.md) — homelab cluster overview.
- [`cluster/k8s-bstrp/docs/README.md`](./k8s-bstrp/docs/README.md) — our
  kubeadm playbook, the other half of this contrast.
- [`cluster/k8s-bstrp/docs/QA.md`](./k8s-bstrp/docs/QA.md) — bootstrap
  Q&A.
- [`argo-applications/README.md`](../layers/argo-applications/README.md) — selector patterns A/B/C/D and
  the service catalog they drive.
- [`observability/README.md`](../layers/observability/README.md) — where the
  mgmt-observability cluster fits into the estate, post-rebuild.
- **EKS Blueprints**: https://aws-ia.github.io/terraform-aws-eks-blueprints/
- **EKS Best Practices Guide**: https://aws.github.io/aws-eks-best-practices/
- **Karpenter docs**: https://karpenter.sh/docs/
- **Crossplane Providers for AWS**: https://marketplace.upbound.io/providers/upbound/provider-family-aws
