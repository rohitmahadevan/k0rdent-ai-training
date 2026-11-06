# Lab 4.1: Hosted Control Plane (k0smotron-Backed `aws-hosted-cp` Template)

**Duration:** 1.5 hours (~45 min active, ~30-40 min waiting for provisioning)
**Type:** Hands-on Lab

> **Validation note:** Validated end-to-end on **k0rdent Enterprise 1.3.1**
> (which ships `aws-hosted-cp-1-0-29`, bundling k0s `v1.35.1+k0s.1`) on
> 2026-06-30. Your install may ship a different revision — discover it in Part 1
> with `kubectl get clustertemplates -n kcm-system | grep aws-hosted-cp`. See the
> **Root Cause & Fix** callout below: revisions whose bundled k0s is older than
> `v1.35.2` / `v1.34.5` / `v1.33.9` need the k0s-version pin from Part 3.

> ## 🩹 ROOT CAUSE & FIX — the hosted CP needs a patched k0s version
>
> **History:** Earlier validation (2026-05-06, against `aws-hosted-cp-1-0-21`)
> hit a persistent `CrashLoopBackOff` on the hosted CP pods
> (`kmc-<cluster-name>-N`):
>
> ```
> Error: failed to start cluster components: can't start ExtensionsReconciler,
> helm CRD is not registered: no matches for kind "Chart" in group "helm.k0sproject.io"
> ```
>
> **Root cause (now understood):** this is **[k0s issue #7832](https://github.com/k0sproject/k0s/issues/7832)**,
> a bootstrap race — *not* a k0rdent/template problem. At controller startup k0s
> applies the `charts.helm.k0sproject.io` CRD asynchronously while the
> `ExtensionsReconciler` (which installs the CCM helm chart) polls for that CRD
> with a hard cap of **10 attempts** (the `#1…#10` lines in the log) and then
> fatally exits. In a **single-pod k0smotron control plane** leader election is
> immediate, so the race is **100% reproducible** — which is exactly why only
> hosted CPs hit it while etcd stays healthy. Fixed upstream by
> [k0s PR #7217](https://github.com/k0sproject/k0s/pull/7217)
> (first in **k0s v1.33.9 / v1.34.5 / v1.35.2**; **never** in the 1.32 line — EOL).
>
> **Why a stock install still breaks:** k0rdent Enterprise **1.3.1** ships
> `aws-hosted-cp-1-0-29`, whose chart pins k0s **`v1.35.1+k0s.1`** — one patch
> release *before* the fix. So the crashloop reproduces out of the box.
>
> **The fix (validated live 2026-06-30):** pin the hosted CP's k0s to
> **`v1.35.4+k0s.0`** — a build that contains the #7217 fix **and** is present in
> the Mirantis Enterprise registry. (The upstream-minimal `v1.35.2` is *not* in
> that registry — it only carries shipped builds `v1.35.1` and `v1.35.4` — so
> pinning `v1.35.2` yields `ErrImagePull`. Use `v1.35.4+k0s.0`.) The hosted CP
> k0s version comes from the template's `K0smotronControlPlane.spec.version`
> (chart value `k0s.version`), **independent of the management cluster's k0s**.
> Part 3 shows two ways to apply the pin. With it, all three CP replicas and etcd
> come up `Running` with no crashloop, and the cluster reaches `Ready`.

---

## Lab Navigation

| Track | Tier | Duration |
|-------|------|----------|
| Foundation | Required | 1.5 hours |

### Week 4 Learning Paths

```
FOUNDATION (start here)        NETWORKING       STORAGE         OPERATIONS              TEMPLATES
4.1 -> 4.2 -> 4.3              4.4              4.5             4.6 -> 4.7              4.8
^
YOU ARE HERE
```

| Previous | Current | Next |
|----------|---------|------|
| [Week 3 (VMaaS)](../../week-3-vmaas/) | **Lab 4.1 — Hosted Control Plane** | [Lab 4.4 — Cilium + Multus](lab-4.4-cilium-multus.md) (4.2/4.3 in development — see the [Week 4 lab map](../README.md)) |

---

## Table of Contents

- [Overview](#overview)
- [Learning Objectives](#learning-objectives)
- [Prerequisites](#prerequisites)
- [Cost Considerations](#cost-considerations)
- [Architectural Decision Frame](#architectural-decision-frame)
- [Part 1: Pre-flight (~10 min)](#part-1-pre-flight-10-min)
- [Part 2: Discover Network Info from Terraform Outputs (~5 min)](#part-2-discover-network-info-from-terraform-outputs-5-min)
- [Part 3: Create the ClusterDeployment (~10 min)](#part-3-create-the-clusterdeployment-10-min)
- [Part 4: Monitor Provisioning (~30 min, mostly waiting)](#part-4-monitor-provisioning-30-min-mostly-waiting)
- [Part 5: Access the Hosted Cluster (~5 min)](#part-5-access-the-hosted-cluster-5-min)
- [Part 6: Compare Footprint vs the Embedded CP from Lab 1.5 (~10 min)](#part-6-compare-footprint-vs-the-embedded-cp-from-lab-15-10-min)
- [Part 7: Failure-Domain Walkthrough (~10 min)](#part-7-failure-domain-walkthrough-10-min)
- [Cleanup (~5 min)](#cleanup-5-min)
- [Verification Checklist](#verification-checklist)
- [Deliverable](#deliverable)
- [Troubleshooting](#troubleshooting)
- [Key Takeaways](#key-takeaways)
- [Next Lab](#next-lab)
- [References](#references)

---

## Overview

This lab provisions a **hosted control plane cluster** by selecting the
`aws-hosted-cp-X-Y-Z` flavor from the k0rdent `ClusterTemplate` catalog. The
API is identical to Lab 1.5's `ClusterDeployment` — only the `template` name
and a few config fields change. Under the hood, the template expands to a
CAPI `Cluster` + `K0smotronControlPlane`, and the control plane runs as
StatefulSet pods on your management cluster instead of on dedicated EC2.

The pedagogical goal is not "deploy k0smotron." It is to **feel the
architectural tradeoff** by running an embedded-CP cluster (`managed-cluster-01`
from Lab 1.5) and a hosted-CP cluster side-by-side, then measuring the cost,
footprint, and failure-domain differences.

> **Reuse:** This lab reuses the management cluster's VPC, subnets, and worker
> security group from `lab-infrastructure/terraform/environments/student-lab`.
> No new networking is provisioned.

---

## Learning Objectives

By completing this lab, you will be able to:

- [ ] Discover the available `aws-hosted-cp-X-Y-Z` template version in your k0rdent install
- [ ] Compose an `aws-hosted-cp` `ClusterDeployment` using existing terraform-provisioned network resources
- [ ] Identify the differences in `config` schema between `aws-standalone-cp` and `aws-hosted-cp`
- [ ] Monitor a hosted-CP provisioning sequence and locate the control-plane pods on the management cluster
- [ ] Measure the resource and cost difference between hosted and embedded control planes
- [ ] Defend, in writing, when each family is the right choice for a fleet

---

## Prerequisites

**Required from earlier weeks:**

- Labs 1.1-1.5 complete
- Lab 1.5's `managed-cluster-01` (an `aws-standalone-cp` cluster) **still running** — this is your comparison baseline
- `aws-cluster-identity-cred` exists in `kcm-system` (created in Lab 1.3)
- `AWSClusterStaticIdentity` `aws-cluster-identity` exists with `kcm-system` allowed (see Lab 1.3 / 1.5)
- `kubectl` pointed at the **management cluster**

**Tools and configuration:**

- `kubectl` v1.32+
- `terraform` v1.8+ (initialized against `lab-infrastructure/terraform/environments/student-lab`) — used for output discovery
- `jq` (for parsing terraform output)
- AWS CLI v2 (for SSO refresh if needed)
- `AWS_REGION` exported, matching your management cluster's region

**Environment outputs you will need:**

This lab uses these terraform outputs from `student-lab`:
- `vpc_id`
- `public_subnet_ids`, `private_subnet_ids`
- `availability_zones`
- `k8s_cluster_security_group_id`
- `nat_gateway_ip`

> If you provisioned your lab environment **before** Week 4 was published, run
> `terraform apply -refresh-only` in the `student-lab` directory to surface the
> new network outputs. They are additive and safe.

---

## Cost Considerations

> **Important:** This lab provisions real AWS infrastructure that incurs costs.
>
> - **Estimated AWS cost during the lab:** ~$0.10/hour for the hosted-cp cluster's worker(s)
> - You will *also* keep `managed-cluster-01` from Lab 1.5 running during this lab (~$0.15/hour) for the side-by-side comparison
> - Combined cost: ~$0.25/hour while running both
> - Clean up the hosted-cp cluster at the end of this lab; you can delete `managed-cluster-01` afterwards if no other Week 4 labs need it immediately

---

## Architectural Decision Frame

> **Predict before you start.** Write your guesses down — you will grade them in Parts 6-7.
>
> 1. "I expect the hosted CP to use approximately ___ MB of memory on the management cluster."
> 2. "I expect the hosted-CP cluster to be ___ % cheaper than the embedded-CP cluster from Lab 1.5."
> 3. "If the management cluster goes down, I expect existing workload pods on the hosted cluster to ___ (keep running / get evicted / fail health checks)."
> 4. "I expect the diff between the two `ClusterDeployment` manifests to be roughly ___ fields."

---

## Part 1: Pre-flight (~10 min)

### Verify Management Cluster

```bash
kubectl config current-context
kubectl get pods -n kcm-system | head -5
```

Expected: pods in kcm-system Running.

### Verify the Existing Lab 1.5 Baseline

```bash
# managed-cluster-01 (the embedded-cp cluster) must be Ready for the comparison
kubectl get clusterdeployment managed-cluster-01 -n kcm-system

kubectl get clusterdeployment managed-cluster-01 -n kcm-system \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' && echo
```

If `managed-cluster-01` is missing or not Ready, redo Lab 1.5 first — Parts 6
and 7 of this lab depend on it being live for direct comparison.

### Verify AWS Credential and Identity

```bash
# Both must exist (created in Lab 1.3)
kubectl get credentials.k0rdent.mirantis.com -n kcm-system aws-cluster-identity-cred
kubectl get awsclusterstaticidentity aws-cluster-identity
```

If `awsclusterstaticidentity` has restrictive `allowedNamespaces`, ensure
`kcm-system` is permitted (Lab 1.5 covers this fix).

### Discover the `aws-hosted-cp` Template Version

```bash
# List ALL templates so you can see the family of available CPs
kubectl get clustertemplates -n kcm-system

# Filter for the hosted-cp family
kubectl get clustertemplates -n kcm-system | grep aws-hosted-cp
```

Note the **exact** template name (e.g., something like `aws-hosted-cp-0-9-2`).
You will reference it in Part 3 as `HOSTED_CP_TEMPLATE`.

```bash
# Save it as a shell variable for the rest of the lab
HOSTED_CP_TEMPLATE="aws-hosted-cp-X-Y-Z"   # <-- replace with what you saw above
echo "Using template: $HOSTED_CP_TEMPLATE"
```

> **Why discover instead of hardcode?** Template versions move with k0rdent
> Enterprise releases. Hardcoding a version in this lab would rot. Discovery
> is a real architect skill — every Week 5 lab does the same.

### (Optional) Inspect the Template

```bash
kubectl get clustertemplate -n kcm-system "$HOSTED_CP_TEMPLATE" -o yaml \
  | head -80
```

Look at `spec.helm.chartSpec` to see which Helm chart this template wraps —
that is the bridge between k0rdent's catalog and k0smotron's CRDs.

---

## Part 2: Discover Network Info from Terraform Outputs (~5 min)

The `aws-hosted-cp` template requires you to bring an existing VPC, subnets,
and security group. We reuse the management cluster's VPC.

### Method A: Direct Terraform Output (preferred)

```bash
cd lab-infrastructure/terraform/environments/student-lab

# Pull each value
VPC_ID=$(terraform output -raw vpc_id)
PUBLIC_SUBNETS=$(terraform output -json public_subnet_ids)
PRIVATE_SUBNETS=$(terraform output -json private_subnet_ids)
AZS=$(terraform output -json availability_zones)
NAT_GW_IP=$(terraform output -raw nat_gateway_ip)
WORKER_SG_ID=$(terraform output -raw k8s_cluster_security_group_id)

# Sanity-check
echo "VPC:        $VPC_ID"
echo "Public:     $PUBLIC_SUBNETS"
echo "Private:    $PRIVATE_SUBNETS"
echo "AZs:        $AZS"
echo "NAT GW IP:  $NAT_GW_IP"
echo "Worker SG:  $WORKER_SG_ID"
```

### Method B: Pull from S3 State (if terraform is not initialized locally)

```bash
ENGINEER_ID="<your-engineer-id>"      # same one used with lab-provision.sh
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET="k0rdent-lab-${ENGINEER_ID}-${ACCOUNT_ID}-${AWS_REGION}"

aws s3 cp "s3://${BUCKET}/terraform.tfstate" - | jq '.outputs |
  {
    vpc_id: .vpc_id.value,
    public_subnet_ids: .public_subnet_ids.value,
    private_subnet_ids: .private_subnet_ids.value,
    availability_zones: .availability_zones.value,
    k8s_cluster_security_group_id: .k8s_cluster_security_group_id.value,
    nat_gateway_ip: .nat_gateway_ip.value
  }'
```

### Discover Per-Subnet Metadata Needed by the Template

The `aws-hosted-cp` template wants `isPublic`, `routeTableId`, and (for public
subnets) `natGatewayID` per subnet entry. Fetch them from AWS:

```bash
# Public subnets — discover their route table IDs and the NAT gateway in each AZ
aws ec2 describe-route-tables --region "$AWS_REGION" \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[].{RTBId:RouteTableId,Subnets:Associations[].SubnetId,Routes:Routes[?DestinationCidrBlock==`0.0.0.0/0`].{NAT:NatGatewayId,IGW:GatewayId}}' \
  --output json | tee /tmp/route-tables.json

# NAT gateway ID(s) in this VPC
NAT_GW_ID=$(aws ec2 describe-nat-gateways --region "$AWS_REGION" \
  --filter "Name=vpc-id,Values=$VPC_ID" \
  --query 'NatGateways[?State==`available`].NatGatewayId' --output text)
echo "NAT Gateway ID: $NAT_GW_ID"
```

Save these — you will paste subnet IDs, AZs, route table IDs, and the NAT
gateway ID into the manifest below.

---

## Part 3: Create the ClusterDeployment (~10 min)

Compose the manifest. The schema differs from Lab 1.5's `aws-standalone-cp`:
no `controlPlaneNumber`, no `controlPlane.instanceType`, but `vpcID` /
`subnets` / `securityGroupIDs` are required.

### Author the Manifest

Replace the `<>` placeholders with values from Part 2:

```bash
cat > /tmp/managed-cluster-02-hosted.yaml <<EOF
apiVersion: k0rdent.mirantis.com/v1beta1
kind: ClusterDeployment
metadata:
  name: managed-cluster-02-hosted
  namespace: kcm-system
  labels:
    environment: training
    cp-mode: hosted
spec:
  template: ${HOSTED_CP_TEMPLATE}
  credential: aws-cluster-identity-cred
  dryRun: false
  cleanupOnDeletion: true
  config:
    # The canonical k0rdent docs use {{.metadata.name}} here — keep it equal to
    # this ClusterDeployment's metadata.name so the underlying CAPA AWSCluster
    # resource name stays self-consistent.
    managementClusterName: managed-cluster-02-hosted
    region: ${AWS_REGION}
    publicIP: true
    vpcID: ${VPC_ID}
    # NOTE: aws-hosted-cp has NO `controlPlaneNumber` field (unlike Lab 1.5's
    # aws-standalone-cp). The k0smotron CP pods AND etcd each run 3 replicas,
    # fixed by the chart — HA persistence is built in. Only `workersNumber` is
    # tunable here.
    workersNumber: 1
    instanceType: t3.medium
    rootVolumeSize: 32
    # IMPORTANT: pin a Ubuntu worker AMI. The aws-hosted-cp chart defaults the
    # worker image to Amazon Linux 2, on which the k0smotron worker bootstrap
    # (cloud-init) does not reliably execute — the EC2 launches but k0s never
    # installs, so the node never joins (no CSR on the child API, konnectivity
    # logs "No agent available", ClusterDeployment stuck at WorkersAvailable).
    # Use the same Ubuntu 22.04 image family as the mgmt node. Discover it with:
    #   aws ec2 describe-images --owners 099720109477 --region $AWS_REGION \
    #     --filters 'Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*' \
    #     --query 'sort_by(Images,&CreationDate)[-1].ImageId' --output text
    amiID: <ubuntu-22.04-ami-id-for-your-region>
    securityGroupIDs:
      - ${WORKER_SG_ID}
    subnets:
      - id: <public-subnet-id-1>
        availabilityZone: <az-1>
        isPublic: true
        natGatewayID: ${NAT_GW_ID}
        routeTableId: <public-route-table-id>
      - id: <private-subnet-id-1>
        availabilityZone: <az-1>
        isPublic: false
        routeTableId: <private-route-table-id>
    clusterIdentity:
      name: aws-cluster-identity
      # `kind` is REQUIRED by the aws-hosted-cp chart schema. Lab 1.3 creates an
      # AWSClusterStaticIdentity named aws-cluster-identity.
      kind: AWSClusterStaticIdentity
    clusterLabels:
      environment: training
      cp-mode: hosted
    # ── THE FIX (k0s #7832) ────────────────────────────────────────────────
    # Pin the hosted CP's k0s to a build that contains the #7217 fix AND exists
    # in your registry. On Enterprise 1.3.1 the template default is the BUGGY
    # v1.35.1+k0s.1; override it here. This config key passes straight through to
    # the chart's k0s.version value (ClusterDeployment.spec.config is
    # x-kubernetes-preserve-unknown-fields). If your discovered template already
    # bundles k0s >= v1.35.2 / v1.34.5 / v1.33.9, you can omit this block.
    k0s:
      version: v1.35.4+k0s.0
EOF
```

> **Two ways to apply the pin.** The inline `config.k0s.version` above is the
> one-line fix and is what this lab validated end-to-end. The *officially
> supported* alternative — when you want the pinned version baked into a
> catalog flavor rather than per-deployment — is a **forked ClusterTemplate**;
> a turnkey helper is provided at
> [`lab-infrastructure/scripts/lab-4.1-fork-hosted-cp-template.sh`](../../../lab-infrastructure/scripts/lab-4.1-fork-hosted-cp-template.sh),
> and Lab 4.8 generalizes this into flavor design.

> **AWS identity permissions.** Your `aws-cluster-identity` credential must allow
> `ec2:DescribeDhcpOptions` (in addition to the usual CAPA permissions). CAPA
> v2.10.0 reads the VPC's DHCP option set during worker provisioning and
> **nil-panics** if that call is denied — the worker EC2 launches but never
> registers, and the cluster hangs at `WorkersAvailable`. The standard
> `clusterawsadm`/Lab 1.3 credentials include this; a minimal hand-rolled policy
> may not.

### Validate Before Apply

```bash
# Dry-run YAML validation
kubectl apply --dry-run=client -f /tmp/managed-cluster-02-hosted.yaml
```

### Apply

```bash
kubectl apply -f /tmp/managed-cluster-02-hosted.yaml

kubectl get clusterdeployment -n kcm-system managed-cluster-02-hosted
```

---

## Part 4: Monitor Provisioning (~30 min, mostly waiting)

The provisioning flow is similar to Lab 1.5 but the control plane appears as
**pods on the management cluster** rather than as new EC2 control-plane nodes.

### Watch the ClusterDeployment

```bash
kubectl get clusterdeployment -n kcm-system managed-cluster-02-hosted -w
# Ctrl-C once Ready
```

### Watch CAPI Resources

```bash
kubectl get cluster -n kcm-system managed-cluster-02-hosted -o yaml | grep -A 10 "status:"
clusterctl describe cluster managed-cluster-02-hosted -n kcm-system
```

### Watch the Control-Plane Pods

This is the moment of truth — the hosted CP appears as pods on the **management
cluster**, not as EC2 instances:

```bash
# k0smotron-managed control plane pods (StatefulSet)
kubectl get statefulset -A | grep -i hosted
kubectl get pods -A | grep managed-cluster-02-hosted
```

You should see something like:

```
kcm-system   kmc-managed-cluster-02-hosted-0   1/1   Running   0   2m
```

The `kmc-` prefix is k0smotron-managed-cluster.

### Watch Worker Provisioning

```bash
# CAPI machines
kubectl get machines -n kcm-system | grep managed-cluster-02-hosted

# Real EC2 workers being created
aws ec2 describe-instances --region "$AWS_REGION" \
  --filters "Name=tag:kubernetes.io/cluster/managed-cluster-02-hosted,Values=owned" \
  --query 'Reservations[].Instances[].[InstanceId,State.Name,PrivateIpAddress]' \
  --output table
```

### Provisioning Stages

```
1. ClusterDeployment created
   ↓
2. KCM expands template → CAPI Cluster + K0smotronControlPlane
   ↓
3. K0smotronControlPlane → StatefulSet on mgmt cluster (CP pods)
   ↓
4. CAPA provider creates worker EC2 instance(s) in the existing VPC
   ↓
5. Workers join the hosted CP via konnectivity
   ↓
6. Cluster Ready
```

Compare this to Lab 1.5's stages — there, *steps 3 and 4 each provisioned EC2
instances*. Here, only step 4 does.

---

## Part 5: Access the Hosted Cluster (~5 min)

```bash
# Wait for Ready
kubectl get clusterdeployment managed-cluster-02-hosted -n kcm-system \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' && echo

# Retrieve kubeconfig (same pattern as Lab 1.5)
kubectl get secret -n kcm-system managed-cluster-02-hosted-kubeconfig \
  -o jsonpath='{.data.value}' | base64 -d > /tmp/managed-cluster-02-hosted.kubeconfig
chmod 600 /tmp/managed-cluster-02-hosted.kubeconfig

# Connect
kubectl --kubeconfig=/tmp/managed-cluster-02-hosted.kubeconfig get nodes
kubectl --kubeconfig=/tmp/managed-cluster-02-hosted.kubeconfig get pods -A | head -20
```

### Smoke Test

```bash
kubectl --kubeconfig=/tmp/managed-cluster-02-hosted.kubeconfig create deployment smoke \
  --image=registry.k8s.io/pause:3.9 --replicas=2

kubectl --kubeconfig=/tmp/managed-cluster-02-hosted.kubeconfig get pods -o wide
```

---

## Part 6: Compare Footprint vs the Embedded CP from Lab 1.5 (~10 min)

This is where the architectural lesson lives.

### Where Does Each CP Run?

```bash
# Lab 1.5 (embedded): CP runs on dedicated EC2 instances
aws ec2 describe-instances --region "$AWS_REGION" \
  --filters "Name=tag:kubernetes.io/cluster/managed-cluster-01,Values=owned" \
            "Name=tag:cluster.x-k8s.io/role,Values=control-plane" \
            "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].[InstanceId,InstanceType]' --output table

# Lab 4.1 (hosted): CP runs as pods on the mgmt cluster — NO dedicated EC2
aws ec2 describe-instances --region "$AWS_REGION" \
  --filters "Name=tag:kubernetes.io/cluster/managed-cluster-02-hosted,Values=owned" \
            "Name=tag:cluster.x-k8s.io/role,Values=control-plane" \
            "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].[InstanceId,InstanceType]' --output table
# (empty — there are no CP EC2 instances for the hosted cluster)
```

### Measure Hosted CP Pod Footprint

```bash
# Find the hosted CP StatefulSet pods
HOSTED_CP_PODS=$(kubectl get pods -A -l 'cluster.x-k8s.io/cluster-name=managed-cluster-02-hosted' \
  -o jsonpath='{range .items[*]}{.metadata.namespace} {.metadata.name}{"\n"}{end}')
echo "$HOSTED_CP_PODS"

# Sum CPU and memory
kubectl top pods -A | grep managed-cluster-02-hosted
```

Record: **___ MB memory, ___ millicores CPU** for the hosted CP on the mgmt cluster.

### Measure Embedded CP Cost

The Lab 1.5 embedded CP runs on a dedicated `t3.medium` (or whatever you specified).
Look up your region's on-demand price — typically ~$0.04/hour for t3.medium = **~$30/month**.

### Cost Comparison Table (fill in your numbers)

| Item | Hosted CP (managed-cluster-02-hosted) | Embedded CP (managed-cluster-01 from Lab 1.5) |
|------|---------------------------------------|------------------------------------------------|
| Where CP runs | StatefulSet pods on mgmt cluster | 1× t3.medium dedicated EC2 |
| Memory used | ~___ MB on mgmt cluster | ~2 GB on dedicated CP node |
| Approx $/month | ~$1-3 (fraction of mgmt cluster) | ~$30/month per CP node |
| Cost ratio | 1× | ~10-30× |

### Diff the Two Manifests

```bash
diff <(kubectl get clusterdeployment -n kcm-system managed-cluster-01 -o yaml | grep -E 'template|controlPlane|worker|vpcID|subnets|securityGroupIDs|instanceType') \
     <(kubectl get clusterdeployment -n kcm-system managed-cluster-02-hosted -o yaml | grep -E 'template|controlPlane|worker|vpcID|subnets|securityGroupIDs|instanceType')
```

Note what changed: same API, different `template`, different `config` schema.

### Threshold Question

> Given a fleet of **N** clusters, at what N does the hosted family clearly win on cost?
> Compute: `(N × $30) > (mgmt cluster headroom for k0smotron pods × cost)`.

Write your N down — it goes in the Deliverable.

---

## Part 7: Failure-Domain Walkthrough (~10 min)

Do **not** actually break your management cluster. Walk through each scenario
on paper.

### Scenario 1: Management Cluster Goes Down

- **`managed-cluster-01` (embedded):** kubelet keeps existing pods alive. CAPI
  reconciliation is paused (controllers are on the mgmt cluster), but
  steady-state traffic continues.
- **`managed-cluster-02-hosted` (hosted):** the kubelet still keeps existing
  pods alive — but the API server itself is on the mgmt cluster. **No new pods
  schedule. No replicaset recovery. No service endpoint propagation.** The
  cluster appears "frozen" from any control perspective.

### Scenario 2: Hosted CP Pod Eviction

- The k0smotron-managed StatefulSet schedules a replacement.
- Persistence model matters: `emptyDir` would lose etcd state; production
  hosted-cp templates use a PVC so state survives. (Check your template's
  persistence default — query `kubectl get k0smotroncontrolplane -A -o yaml`.)

### Scenario 3: Mgmt Cluster Network Partition

- Every hosted CP becomes unreachable from its workers simultaneously.
- One outage spans every hosted cluster in the fleet — the signature failure
  mode of the hosted family.

### Document Your Findings

In a short paragraph, list which scenarios are **worse** with hosted than
embedded, and which are **the same**. This goes in the Deliverable.

---

## Cleanup (~5 min)

Cleanup is dramatically simpler than my previous draft of this lab — there are
no manual EC2 instances or join tokens to revoke. k0rdent owns it all.

```bash
# Delete the hosted cluster (k0rdent + CAPA tear down workers + CP pods)
kubectl delete -f /tmp/managed-cluster-02-hosted.yaml

# Wait for full teardown (~3-5 minutes)
kubectl get clusterdeployment -n kcm-system -w
# Press Ctrl-C once managed-cluster-02-hosted is gone

# Verify no orphan EC2
aws ec2 describe-instances --region "$AWS_REGION" \
  --filters "Name=tag:kubernetes.io/cluster/managed-cluster-02-hosted,Values=owned" \
            "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].InstanceId' --output text
# (empty)

# Remove the local kubeconfig
rm -f /tmp/managed-cluster-02-hosted.kubeconfig /tmp/managed-cluster-02-hosted.yaml
```

> If a subsequent Week 4 lab needs `managed-cluster-01` (the Lab 1.5 cluster),
> leave it running. Otherwise delete it now to stop billing.

---

## Verification Checklist

- [ ] `kubectl get clustertemplates -n kcm-system | grep aws-hosted-cp` returned a valid template
- [ ] Terraform outputs (`vpc_id`, `public_subnet_ids`, etc.) fetched successfully
- [ ] `managed-cluster-02-hosted` `ClusterDeployment` Ready
- [ ] `kubectl get statefulset -A | grep hosted` shows the k0smotron-managed CP StatefulSet
- [ ] No EC2 instances tagged as control-plane for `managed-cluster-02-hosted` (they should not exist)
- [ ] `kubectl --kubeconfig=...` to the hosted cluster works
- [ ] Smoke deployment Running on hosted cluster workers
- [ ] CP footprint measured (memory + CPU on mgmt cluster)
- [ ] Manifest diff between `managed-cluster-01` and `managed-cluster-02-hosted` reviewed
- [ ] Failure-domain analysis written (3 scenarios)
- [ ] Architectural Decision Frame predictions graded
- [ ] Cleanup verified (no orphan EC2, no leftover Kubernetes resources)

---

## Deliverable

A **1-paragraph written rationale** answering:

> "For a customer running 30 small dev clusters and 3 large training clusters,
> which clusters should use the `aws-hosted-cp` template family and which should
> use `aws-standalone-cp`? Defend your split using the cost math from Part 6
> and the failure-domain analysis from Part 7."

Strong answers cite:
- A specific N-threshold from your Part 6 math
- At least one failure-domain consideration from Part 7
- A pre-emptive answer to "what if the customer wants tenant-level upgrade independence?"

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `kubectl get clustertemplates -n kcm-system | grep aws-hosted-cp` returns nothing | The `aws-hosted-cp` template family is not in your k0rdent install | Confirm k0rdent 1.3.x ships hosted-cp templates; check `kubectl get providertemplates -n kcm-system` for the hosted-cp provider |
| YAML rejected with "unknown field natGatewayID" | The shipped CRD validation expects `natGatewayId` (lowercase d) | Swap case on the field in each public subnet entry — the k0rdent docs are internally inconsistent; either case may apply per template version |
| `ClusterDeployment` rejected with "managementClusterName invalid" | Some k0rdent template versions expect a literal AWS context name | Try `managementClusterName: aws` (the literal value from the docs admin/hcp-aws.md page) instead of the cluster's `metadata.name` |
| `terraform output vpc_id` returns empty | Outputs added to outputs.tf but state is older | `cd lab-infrastructure/terraform/environments/student-lab && terraform apply -refresh-only` |
| ClusterDeployment Pending forever | The `aws-cluster-identity` `allowedNamespaces` is restrictive | `kubectl patch awsclusterstaticidentity aws-cluster-identity --type=merge -p '{"spec":{"allowedNamespaces":{}}}'` |
| Workers stuck `Provisioning` | Subnet doesn't have a route to NAT gateway, or `securityGroupIDs` blocks egress | Re-check the per-subnet `routeTableId` and `natGatewayID` values from Part 2; verify the SG allows egress 443 |
| `kmc-managed-cluster-02-hosted-0` Pod Pending | Mgmt cluster lacks resources for the request | Check `kubectl describe pod` — typically it's CPU/memory requests on the mgmt nodes |
| Workers join but `NotReady` | CNI not yet rolled out | Wait 2-3 minutes; check `kubectl --kubeconfig=... -n kube-system get pods` |
| `ExpiredTokenException` in CAPA logs (SSO users) | AWS SSO token expired during the long provisioning wait | Refresh: `aws sso login --profile <p>`; update the secret per Lab 1.3; then `kubectl rollout restart deployment capa-controller-manager -n kcm-system` |
| `kubeconfig` retrieved but `Unable to connect to server` | `service.type: LoadBalancer` not yet provisioned (NLB takes 2-3 min) | Wait; verify NLB exists with `aws elbv2 describe-load-balancers` |
| `kmc-<cluster-name>-N` Pod `CrashLoopBackOff`, logs show `can't start ExtensionsReconciler, helm CRD is not registered ... no matches for kind "Chart"` | The template's bundled k0s predates the [#7217](https://github.com/k0sproject/k0s/pull/7217) fix for [k0s #7832](https://github.com/k0sproject/k0s/issues/7832) (e.g. `v1.35.1+k0s.1` in `aws-hosted-cp-1-0-29`) | **Pin the hosted-CP k0s** to a fixed+available build via `config.k0s.version: v1.35.4+k0s.0` (see Part 3 and the Root Cause & Fix callout). Not awaiting a template revision — fixed by the version pin. |
| etcd pods `Running` but `kmc-<cluster-name>-N` (CP) crashlooping with the helm CRD error | Expected: etcd is independent and comes up first; only the CP container hits the #7832 race | Same fix as the row above — pin `config.k0s.version` to `v1.35.4+k0s.0`. |
| Hosted CP `Running` but cluster stuck at `WorkersAvailable`; worker EC2 exists but never registers; CAPA logs show `panic: nil pointer dereference` in `GetDHCPOptionSetDomainName` | CAPA v2.10.0 nil-panics when its AWS identity is denied `ec2:DescribeDhcpOptions` while reading the VPC's DHCP option set | Ensure your `aws-cluster-identity` credentials allow `ec2:DescribeDhcpOptions`. `clusterawsadm bootstrap`/Lab 1.3 creds include it; a minimal policy may not. |

---

## Key Takeaways

1. **In k0rdent, "hosted vs embedded" is a `ClusterTemplate` choice — not a different API.** You write `ClusterDeployment` either way; only the `template` field changes.
2. **The hosted family's `config` schema requires you to bring an existing VPC.** That's because the workers must reach the management cluster, and the natural answer is to put them in the management cluster's network. Lab 4.1 reuses `lab-infrastructure/terraform`'s outputs for exactly this reason.
3. **A hosted control plane is a few pods on the mgmt cluster, not a few VMs.** Per-cluster CP cost drops 10-30×.
4. **The kubelet keeps existing pods alive even when the API server is down** — both patterns survive a mgmt-cluster outage *for steady state*. Only embedded can self-heal during one.
5. **You typically never write `K0sControlPlane`, `K0smotronControlPlane`, or `k0smotron.io Cluster` directly.** Templates wrap them. Lab 4.8 will let you encode hosted-vs-embedded as a custom flavor decision.

---

## Next Lab

[Lab 4.4 — Cilium + Multus](lab-4.4-cilium-multus.md) — customizes the CNI and adds secondary networks on a k0rdent-managed cluster, building on the template-fork pattern from this lab. (Lab 4.2 — Multi-Provider GPU Cluster — is in development; see the [Week 4 lab map](../README.md).)

---

## References

- k0rdent — [AWS Hosted Control Plane docs](https://github.com/k0rdent/docs/blob/main/docs/admin/hosted-control-plane/hcp-aws.md)
- k0rdent — [`ClusterTemplate` catalog reference](https://github.com/k0rdent/docs/blob/main/docs/quickstarts/quickstart-2-remote.md)
- k0smotron — [Project documentation](https://docs.k0smotron.io/) (architecture under `K0smotronControlPlane`)
- Theory 4.1 — KaaS Architecture, §3.3 Two Template Families (in development, not yet committed)
- Lab 1.5 — [Provision Your First Managed Cluster](../../week-1-foundations/labs/lab-1.5-provision-managed-cluster.md) (embedded-cp baseline)
- Lab Infrastructure — [`lab-infrastructure/terraform/environments/student-lab/outputs.tf`](../../../lab-infrastructure/terraform/environments/student-lab/outputs.tf) (network outputs reused here)
