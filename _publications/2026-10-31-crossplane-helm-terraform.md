---
title: "Terraform, Helm, and Crossplane: Separating the Control Plane from the Data Plane"
collection: publications
permalink: /publications/crossplane-helm-terraform
excerpt: 'One EKS cluster per environment, with the control plane and the data plane split by account, by ownership, and by node pool. Terraform builds the foundation and the IAM fence, Helm packages both the platform API and the service chart, and Crossplane reconciles per-app cloud resources from claims. A dedicated management cluster is a later step the ApplicationSets already leave room for.'
date: 2026-10-31
venue: 'Personal website'
tags:
  - crossplane
  - terraform
  - helm
  - gitops
---
{% raw %}
## Summary

Terraform, Helm, and Crossplane each get the job that matches how they fail.

Terraform applies a stack, records a state file, and is the right owner for objects whose replacement takes the platform down with them: the VPC, the EKS cluster, the IAM roles that fence every other tool, and the Argo CD install that starts GitOps. Crossplane reconciles claims continuously, and is the right owner for the queue, the bucket, the key, and the IRSA role that belong to one service and change on that service's schedule. Helm is the package format on both sides of that line. One chart publishes the platform API (providers, an `EnvironmentConfig`, XRDs, Compositions). Another chart is what a service renders, and it emits a Deployment and Crossplane claims from the same values files.

Those tools share a cluster. Each environment runs one EKS cluster. That cluster is where Argo CD and Crossplane run, and it is also where the workloads run. The control plane and the data plane are separated by AWS account, by which tool owns the object, and by a tainted Karpenter node pool. A management cluster that targets many workload clusters is a later step. The ApplicationSets are already written as that fan-out: they select clusters by the label `tier: application`.

The listings below follow the real layout and replace identifiers. Account IDs, VPC and subnet IDs, OIDC issuers, state-bucket names, and the domain used on internal labels are placeholders. Label keys are shown short; on the cluster they are domain-scoped keys with the same meaning.

## Three lines on a single cluster

Each environment runs one EKS cluster. Argo CD and Crossplane are installed on it, and the workloads run on it too. The separation between control plane and data plane is three concrete lines: which account holds the object, which tool is allowed to change it, and which nodes it runs on.

**Accounts.** A shared AWS account holds the Terraform state bucket, the IAM role used by the PR-driven apply (Atlantis), and shared secrets. Each workload account (the worked example is staging; production is the same shape where that stack is filled in) holds the VPC, the EKS cluster, and the per-app cloud resources. State is central. The state key is the Terragrunt path, the backend uses the S3 lockfile (`use_lockfile`), and there is no DynamoDB lock table. Crossplane's IAM policy explicitly denies access to the state bucket, so the reconciler cannot read or rewrite the foundation's state.

**Ownership.** Terraform owns the long-lived foundation: network, cluster, KMS used by the platform, Secrets Manager containers, the Crossplane IRSA role, the permissions boundary every role Crossplane creates must carry, and Argo CD itself. Crossplane owns the short-lived, per-app resources a service declares: S3 buckets, SQS queues (with DLQ and SNS subscriptions), SNS topics, KMS keys, Secrets Manager secrets, and IRSA roles. The XRD states the same dependency rule the Terraform modules use: a topic exists before a queue subscribes to it, and the subscription is owned by the queue side. The two layers share that convention, each in its own codebase.

**Nodes.** Inside the one cluster, a Karpenter `NodePool` named `platform` carries the taint `workload=platform:NoSchedule`. Argo CD, Crossplane, and the cluster addons tolerate that taint and set a matching node selector. Workloads land on the other pools: `apps`, `apps-spot`, `gpu`, `stateful`, and `general-purpose`. Application namespaces receive the label `workloads=true`, and network policies use that label to keep them apart from the platform namespaces.

```text
shared account                         workload account (one per environment)
  tfstate bucket                         Terraform
  Atlantis role                            VPC, EKS, platform KMS
  shared secrets                           secret containers
        ^                                  Crossplane IRSA role
        |                                  app permissions boundary
        | state (S3, key = stack path)     Argo CD helm_release, once
        |
        +-------------------------------- EKS cluster
                                           NodePool "platform"
                                             taint workload=platform:NoSchedule
                                             Argo CD, Crossplane, addons
                                           NodePools apps, apps-spot, gpu,
                                             stateful, general-purpose
                                             service pods + Crossplane claims
                                               (S3, SQS, SNS, KMS, secrets, IRSA)
```

The practical consequence: a developer can add a queue without a Terraform plan, and cannot replace the VPC by editing a claim. A bad chart sync can still hurt the cluster both planes share. That shared fate is the trade-off the node taint does not remove. It is discussed at the end, along with the step that would remove it.

## How a change moves

Four repositories, four jobs.

| Repo | What it holds |
|---|---|
| `iac-catalog` | Reusable Terraform modules, released by tag |
| `iac-live` | Terragrunt stacks, one directory tree per environment |
| `helm-charts` | The `crossplane` chart (platform API) and the `apps` chart (workloads and claims), published as OCI artifacts |
| `argocd` | The GitOps bootstrap, AppProjects, addon configuration, node pools, network policy |

`iac-catalog` is the module library (`aws/eks`, `aws/vpc`, `aws/iam-role`, the Kubernetes module that installs Argo CD, plus the neighboring modules for DNS, zero trust, and managed data stores). Releases are cut by semantic-release. `iac-live` pins a `catalog_ref` and every unit points at that ref:

```hcl
terraform {
  source = "git::https://github.com/<ORG>/iac-catalog.git//modules/aws/eks?ref=${local.catalog_ref}"
}
```

Live configuration stays thin. A unit under `_units/<provider>/<module>` wraps one catalog module. An environment stack composes units. The cluster stack is two units, the cluster and the Argo CD install, and the cluster unit reads the VPC from the network stack:

```hcl
unit "eks" {
  source = "${local.units_path}/aws/eks"
  path   = "eks"
  values = read_terragrunt_config("${get_terragrunt_dir()}/eks.values.hcl").locals.values
}

unit "argocd" {
  source = "${local.units_path}/kubernetes/argocd"
  path   = "argocd"
  values = read_terragrunt_config("${get_terragrunt_dir()}/argocd.values.hcl").locals.values
}
```

```hcl
dependency "vpc" {
  config_path = "../../../network/workloads/.terragrunt-stack/vpc"
  mock_outputs = {
    vpc_id          = "vpc-000"
    private_subnets = ["subnet-1", "subnet-2", "subnet-3"]
  }
  mock_outputs_allowed_terraform_commands = ["init", "plan", "validate"]
}

inputs = {
  vpc_id     = dependency.vpc.outputs.vpc_id
  subnet_ids = dependency.vpc.outputs.private_subnets
}
```

`plan` works before the VPC exists. `apply` still requires the real dependency outputs. The allow-list is what keeps a mocked id out of a real apply.

Remote state lives only in the shared account:

```hcl
remote_state {
  backend = "s3"
  config = {
    encrypt      = true
    bucket       = "<TFSTATE_BUCKET>"
    key          = "${path_relative_to_include()}/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true
  }
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
}
```

Atlantis plans and applies these stacks from a pull request. After apply, Terraform's job toward the cluster is finished except for the outputs someone has to carry across the hand-off described below. Day-to-day installs of addons and services happen in git, through Argo CD.

## The IAM fence

Crossplane's AWS credentials are an IRSA role Terraform creates, one per environment, assumed by the provider's service account (`crossplane-system` / `crossplane-provider-aws`). The chart annotates that service account via a `DeploymentRuntimeConfig`. The `ProviderConfig` then says `credentials.source: IRSA`. Terraform owns the policy. The chart only receives the role ARN.

The policy is the actual control plane. A trimmed form:

```hcl
policy_json = jsonencode({
  Version = "2012-10-17"
  Statement = [
    {
      Sid      = "IamRolesRequireBoundary"
      Effect   = "Allow"
      Action   = ["iam:CreateRole", "iam:PutRolePolicy"]
      Resource = ["arn:aws:iam::<ACCOUNT_ID>:role/app-*"]
      Condition = {
        StringEquals = {
          "iam:PermissionsBoundary" = "arn:aws:iam::<ACCOUNT_ID>:policy/app-boundary-<ENV>"
        }
      }
    },
    {
      Sid      = "S3BucketsUnderPrefix"
      Effect   = "Allow"
      Action   = "s3:*"
      Resource = ["arn:aws:s3:::app-*", "arn:aws:s3:::app-*/*"]
    },
    {
      Sid    = "SqsQueuesUnderPrefix"
      Effect = "Allow"
      Action = [
        "sqs:CreateQueue",
        "sqs:DeleteQueue",
        "sqs:SetQueueAttributes",
        # remaining provider verbs, still scoped to app-*
      ]
      Resource = "arn:aws:sqs:us-east-1:<ACCOUNT_ID>:app-*"
    },
    {
      Sid      = "DenyKmsKeyDestruction"
      Effect   = "Deny"
      Action   = ["kms:ScheduleKeyDeletion", "kms:DisableKey"]
      Resource = "*"
    },
    {
      Sid      = "DenyTerraformState"
      Effect   = "Deny"
      Action   = "s3:*"
      Resource = ["arn:aws:s3:::<TFSTATE_BUCKET>", "arn:aws:s3:::<TFSTATE_BUCKET>/*"]
    },
  ]
})
```

Read it as a fence with uneven boards, because that unevenness matters.

Roles Crossplane creates must sit under the `app-*` prefix **and** must carry the permissions boundary. Forgetting the boundary on a `Role` in a Composition is an IAM deny. The boundary policy itself is Terraform's, so a claim cannot widen what an application role is allowed to become.

SQS is verb-scoped and prefix-scoped. S3 is prefix-scoped only (`s3:*` on `app-*`). The prefix is doing the blast-radius work for buckets. That is a conscious coarse edge: a composition can do anything S3 allows, to any bucket under the prefix, including one that belongs to another service. The name prefix is a guardrail. It becomes a tenant boundary only if names are unique and nobody reaches sideways.

Two denies sit on top of the allows. Crossplane cannot schedule deletion or disable a KMS key, anywhere. Crossplane cannot touch the Terraform state bucket, so a bug in a composition, or a leaked set of credentials for this role, cannot delete or encrypt the foundation's state. The deny is what makes the account split real. Shared state and workload reconciliation are different IAM principals.

The same prefix shows up in the compositions. A queue's cloud name is `app-<short-name>-<environment>`, built inside the composition from the claim parameter and the environment context. The claim author picks `uploads`. They do not pick the account, the region, or the full AWS name.

## Bootstrap, then step back

Terraform installs Argo CD with a `helm_release` of the upstream chart. The release injects one extra object: an `Application` named `bootstrap`, annotated as a Helm post-install, post-upgrade hook. It points at two directories of the GitOps repo, `bootstrap/` and `appprojects/`, with automated prune and self-heal, destination the same cluster.

```hcl
extraObjects = [{
  apiVersion = "argoproj.io/v1alpha1"
  kind       = "Application"
  metadata = {
    name      = "bootstrap"
    namespace = var.namespace
    annotations = {
      "helm.sh/hook" = "post-install,post-upgrade"
    }
  }
  spec = {
    project = "default"
    sources = [for path in ["bootstrap", "appprojects"] : {
      repoURL        = var.bootstrap_repo_url
      targetRevision = var.bootstrap_target_revision
      path           = path
      directory      = { include = "*.yaml" }
    }]
    destination = {
      server    = "https://kubernetes.default.svc"
      namespace = var.namespace
    }
    syncPolicy = {
      automated = { prune = true, selfHeal = true }
    }
  }
}]
```

The same values register the in-cluster cluster secret and label it. Those labels are the contract the generators match on: `tier=application`, the environment name, and the account id. Repository credentials for Argo CD come from the secret store. They are wired in this release and they do not belong in git.

Platform pods are pinned at install time, before any ApplicationSet exists:

```hcl
controller = {
  nodeSelector = { workload = "platform" }
  tolerations = [{
    key      = "workload"
    operator = "Equal"
    value    = "platform"
    effect   = "NoSchedule"
  }]
}
```

Once the hook has synced, Terraform is out of the path for addons and apps. Deleting the Helm release would delete the bootstrap, which is why that release is treated as foundation and changed rarely. Everything it created is owned by Argo CD after the first successful sync.

Two ApplicationSets in `bootstrap/` do the rest. Both are a matrix. The first generator is the cluster list. The second is "what should run there."

Addon discovery crosses clusters labeled `tier: application` with every `addons/*/config.yaml`:

```yaml
spec:
  goTemplate: true
  generators:
    - matrix:
        generators:
          - clusters:
              selector:
                matchLabels:
                  tier: application
          - git:
              repoURL: https://github.com/<ORG>/argocd.git
              revision: main
              files:
                - path: "addons/*/config.yaml"
  template:
    metadata:
      name: '{{ .addonDir }}-{{ index .metadata.labels "cluster" }}'
      annotations:
        argocd.argoproj.io/sync-wave: '{{ dig "wave" "5" . }}'
    spec:
      project: addons
      sources:
        - repoURL: "{{ .chartRepo }}"
          chart: "{{ .chartName }}"
          targetRevision: "{{ .chartVersion }}"
          helm:
            valueFiles:
              - "$values/addons/{{ .addonDir }}/values/default.yaml"
              - '$values/addons/{{ .addonDir }}/values/{{ index .metadata.labels "cluster" }}.yaml'
            ignoreMissingValueFiles: true
        - repoURL: https://github.com/<ORG>/argocd.git
          targetRevision: main
          ref: values
      destination:
        server: "{{ .server }}"
        namespace: "{{ .namespace }}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - ServerSideApply=true
```

A small file turns a directory into an addon. The upstream Crossplane chart and the in-house platform chart are two of them:

```yaml
# addons/crossplane/config.yaml
addonDir: crossplane
chartRepo: https://charts.crossplane.io/stable
chartName: crossplane
chartVersion: "2.4.2"
namespace: crossplane-system
wave: "1"
---
# addons/crossplane-platform/config.yaml
addonDir: crossplane-platform
chartRepo: ghcr.io/<ORG>/helm-charts
chartName: crossplane
chartVersion: "0.1.*"
namespace: crossplane-system
wave: "3"
```

The upstream chart is pinned to an exact version. The chart this platform publishes floats inside `0.1.*`. A patch release of the platform API rolls forward on the next sync, with no edit to the GitOps repo. A bad publish rolls forward the same way, onto every cluster the generator matches. That is the trade-off of a floating constraint, and it is why the control-plane chart from upstream stays exact.

App discovery uses the same cluster selector and, as the second generator, the GitHub organization. Any repository carrying the label `argocd-app` becomes an Application. The chart is `apps` from the same OCI registry, constraint `0.2.*`. Value files are read from that service repository's `deploy/` directory, and missing per-environment files are ignored, so a service that only has `deploy/values.yaml` and `deploy/resources.yaml` is valid. Argo CD injects the environment name and the account id from the cluster labels, which means a service author never pastes an account id into git. The Application's project is `apps`, the destination namespace is the repository name, and the namespace is labeled `workloads=true`.

Addon Applications and service Applications land in different AppProjects. That is the seam where a service repo stays on workloads and claims, and the addon repo stays on CRDs, providers, and cluster-scoped policy.

## EnvironmentConfig is the contract

Compositions need the account id, the region, the VPC, the private subnet ids, the OIDC issuer, and the permissions-boundary ARN. None of those belong in a claim, and none of them should be hardcoded in a Composition. Crossplane is denied the state bucket, so it also cannot look them up.

The bridge is a values file in the GitOps repo, filled from Terraform outputs, rendered by the platform chart into an `EnvironmentConfig` named `aws-network`:

```yaml
# addons/crossplane-platform/values/<env>.yaml
aws:
  environment: staging
  region: us-east-1
  accountId: "<ACCOUNT_ID>"
  provider:
    iamRoleArn: "arn:aws:iam::<ACCOUNT_ID>:role/app-crossplane-staging"
  network:
    vpcId: <VPC_ID>
    privateSubnetIds: [<SUBNET_A>, <SUBNET_B>, <SUBNET_C>]
  eksClusterName: <EKS_CLUSTER_NAME>
  oidcProvider: oidc.eks.us-east-1.amazonaws.com/id/<OIDC_ID>
```

```yaml
apiVersion: apiextensions.crossplane.io/v1beta1
kind: EnvironmentConfig
metadata:
  name: aws-network
data:
  accountId: "{{ required "aws.accountId is required" .Values.aws.accountId }}"
  environment: {{ required "aws.environment is required" .Values.aws.environment }}
  region: {{ .Values.aws.region }}
  oidcProvider: {{ required "aws.oidcProvider is required" .Values.aws.oidcProvider }}
  permissionsBoundaryArn: {{ .Values.aws.permissionsBoundaryArn | default (printf "arn:aws:iam::%s:policy/app-boundary-%s" (.Values.aws.accountId | toString) .Values.aws.environment) | quote }}
  vpcId: {{ required "aws.network.vpcId is required" .Values.aws.network.vpcId }}
  privateSubnetIds:
    {{- toYaml .Values.aws.network.privateSubnetIds | nindent 4 }}
```

`required` fails the Helm render when the hand-off was not filled in. That is a better failure than a Composition that creates queues in an empty region and reports them ready. The boundary ARN has a conventional default so the common name does not have to be repeated, and an explicit value still wins when an environment differs.

Every composition pipeline starts by loading that object by name:

```yaml
- step: environmentConfigs
  functionRef:
    name: crossplane-contrib-function-environment-configs
  input:
    apiVersion: environmentconfigs.fn.crossplane.io/v1beta1
    kind: Input
    spec:
      environmentConfigs:
        - type: Reference
          ref:
            name: aws-network
```

The Go template then reads `index .context "apiextensions.crossplane.io/environment"`. Account, region, and environment are context. The claim carries only what is specific to the resource.

This file is a copy of Terraform outputs, reviewed in git, and it changes only when someone edits it. Changing a private subnet is a stack change and an edit to this values file. Skip the second edit and the cluster stays healthy, existing resources stay where they are, and the next claim is built against a network the VPC stack no longer has. The deny on the state bucket is what forces this copy to exist. The copy is what can go stale. Both are properties of the design.

## Helm on both sides of the API

The platform chart (`helm-charts/charts/crossplane`) is the API. Its values name the provider package version (`v2.8.1` of the Upbound family, one provider per AWS service), the function versions (`function-environment-configs` v0.9.0, `function-go-templating` v0.13.0, `function-auto-ready` v0.6.9), and the per-environment network block above. Templates render:

- `Provider` objects, each referencing a runtime config that carries the IRSA annotation
- the `ProviderConfig` (`credentials.source: IRSA`)
- the `EnvironmentConfig`
- an XRD and a Composition per resource kind (IRSA, KMS, S3, Secrets Manager, SNS, SQS)

The service chart (`helm-charts/charts/apps`) is the client of that API. The same render produces the Deployment, Service, HPA, PDB, HTTPRoute, GRPCRoute, NetworkPolicy, ExternalSecret, and ServiceAccount, and it produces one claim per entry under `infrastructure:`. A library chart holds the IRSA and IAM fragments those templates share, so each resource template does not reimplement the role shape.

Shipping Compositions inside a Helm chart has a sharp edge. The Composition is a Go template, stored as a string, evaluated later by `function-go-templating`. Helm evaluates `{{ }}` at render time. Left alone, Helm will consume the Composition's template and write a static, empty manifest into the cluster. The chart escapes every Go expression so Helm treats it as text:

```text
# in the chart; Helm emits the inner expression unchanged
{{`{{- $params := .observed.composite.resource.spec.parameters }}`}}
{{`{{- $env := index .context "apiextensions.crossplane.io/environment" }}`}}

# stored on the Composition after Helm renders
{{- $params := .observed.composite.resource.spec.parameters }}
{{- $env := index .context "apiextensions.crossplane.io/environment" }}
```

The first time a line is added without the escape, the chart installs cleanly. Claims then render with blank names, or the function throws, depending on which expression Helm ate. The failure shows up on the claim, not on the addon sync that introduced it.

## What a service author writes

A service repository carries `deploy/values.yaml` for the workload and `deploy/resources.yaml` for cloud resources. This is the whole data-plane API for a bucket:

```yaml
# deploy/values.yaml
image:
  repository: ghcr.io/<ORG>/files
  tag: 0.1.0
ports:
  - { name: http, containerPort: 9091 }
  - { name: grpc, containerPort: 9090, appProtocol: kubernetes.io/h2c }
httpRoute:
  enabled: true
  gatewayName: internal
  port: 9091
resources:
  requests: { cpu: 100m, memory: 128Mi }
  limits: { memory: 256Mi }
podDisruptionBudget:
  enabled: true
  maxUnavailable: 1
---
# deploy/resources.yaml
infrastructure:
  s3:
    buckets:
      - name: uploads
        versioning: true
        tags:
          service: files
```

A queue is the same shape. The chart fails the render if `name` is missing, then emits a claim. The claim's Kubernetes name is the service name plus the short name, truncated to 63 characters. The AWS name is a different string, built later in the composition.

```yaml
{{- range $queue := .Values.infrastructure.sqs.queues }}
{{- if not $queue.name }}
{{- fail "infrastructure.sqs.queues[].name is required" }}
{{- end }}
---
apiVersion: crossplane.example.com/v1alpha1
kind: AWSSQSQueue
metadata:
  name: {{ printf "%s-%s" (include "apps.name" $) $queue.name | trunc 63 | trimSuffix "-" }}
spec:
  parameters:
    name: {{ $queue.name | quote }}
    visibilityTimeoutSeconds: {{ $queue.visibilityTimeoutSeconds | default 30 }}
    {{- if hasKey $queue "dlq" }}
    dlq:
      maxReceiveCount: {{ (default dict $queue.dlq).maxReceiveCount | default 5 }}
    {{- end }}
    {{- with $queue.subscriptions }}
    subscriptions:
      {{- range $sub := . }}
      - topic: {{ $sub.topic | quote }}
        rawMessageDelivery: {{ if hasKey $sub "rawMessageDelivery" }}{{ $sub.rawMessageDelivery }}{{ else }}false{{ end }}
      {{- end }}
    {{- end }}
{{- end }}
```

The XRD (`XAWSSQSQueue`, claim kind `AWSSQSQueue`) accepts a short name, visibility and retention, an optional DLQ (`maxReceiveCount`, default 5), and subscriptions. Each subscription names a topic, a raw-delivery flag, and an optional filter policy. The schema description records the dependency rule: subscriptions live on the queue so the edge stays one-directional, topic first, queue after, matching the Terraform modules. Status surfaces the queue ARN, the queue name, and the DLQ ARN once the composition writes them.

The Composition is Pipeline mode with three steps: load `aws-network`, render with `function-go-templating`, then `function-auto-ready`. The render step builds both names from context, so the claim cannot escape the prefix:

```text
{{`{{- $name := $params.name | default $claimName }}`}}
{{`{{- $queueName := printf "app-%s-%s" $name $env.environment }}`}}
{{`{{- $dlqName := printf "app-%s-dlq-%s" $name $env.environment }}`}}
```

From there the template emits a `Queue`, a DLQ `Queue` when `parameters.dlq` is set, a `TopicSubscription` per subscription, and a `QueuePolicy` that allows `sns.amazonaws.com` to `SendMessage` from those topic ARNs. The queue URL on the policy is selected with `matchControllerRef` plus a `role: main` label, so the policy binds to the queue this claim owns and not to a second queue that happens to share a name fragment. Tags include `managed-by: crossplane`, which is what distinguishes these objects from anything Terraform still creates under the same prefix.

### Two defaults for one flag

`rawMessageDelivery` is the bug that shows why the chart default is the one that has to be right.

The XRD schema defaults the flag to `true`. Raw delivery is what most new subscribers want: the body is the message the publisher sent. The service chart defaults it to `false`, and it always writes the field. The consumer helper in these services parses the SNS envelope (type, message id, the inner message). With raw delivery on, that helper receives the bare payload, fails the parse, and the message is retried until it lands on the DLQ. The queue looks subscribed. The DLQ is the only symptom.

Because the chart sets the field on every claim, the XRD default never applies to resources created through `deploy/resources.yaml`. A claim written by hand, omitting the field, still gets `true` from the schema. There are two APIs. The one service authors call is the chart, and its default was learned from a DLQ. The schema default describes the other caller. When they disagree, document which one the platform actually supports, and test the chart path. A unit test that only renders the XRD will bless the default that production does not use.

## Sync waves

The first apply of this stack fails if everything is synced in one wave. Providers are not installed yet when the `ProviderConfig` is applied. XRDs are not served yet when a claim appears. Argo CD reports a comparison error, someone clicks Sync again, and the second attempt works. The waves exist so the first attempt is the one that works.

Waves on an Application order Applications against each other. Waves on a resource order objects inside one Application. Both are in use, and they are different counters.

| What | Wave | Waits for |
|---|---|---|
| Application: upstream `crossplane` chart | 1 | nothing platform-specific; this installs the CRDs and core controllers |
| Application: `crossplane-platform` chart | 3 | the core chart |
| `Provider` packages inside the platform chart | 1 | sibling objects in later waves of the same Application |
| `ProviderConfig` and `EnvironmentConfig` | 5 | provider packages |
| XRDs and Compositions | 10 | the environment object and the functions they reference |

Claims are not part of the platform chart. They arrive with the service Application, after the XRD is established. `SkipDryRunOnMissingResource` on the provider objects keeps Argo CD from failing a dry run against a CRD the provider is about to install. Server-side apply is on for the addon Applications and off for the provider objects that reject it. Those two annotations are as load-bearing as the wave numbers. Copying the wave table without them brings the "sync twice" failure back.

## Lessons

**The tool follows the blast radius.** A claim that reconciles every few minutes is a good fit for a queue and a bad fit for a VPC. Terraform's plan is slow, reviewed, and stateful, which is what you want for the network, the cluster, the IAM fence, and the Argo CD bootstrap. The cost is lead time: a new AWS primitive is a module and a composition, not a checkbox. That lead time is the point. Self-service starts after the API exists.

**The fence is the policy, and the boards are uneven.** Prefix plus a mandatory permissions boundary plus two denies (KMS destruction, the state bucket) is the real separation between Crossplane and the foundation. SQS is locked down by verb. S3 is locked down by name. Reviewing a new composition means asking which of those two patterns the new service follows, and whether a sideways call to a neighbor's bucket should be possible. The boundary on `iam:CreateRole` is the statement to protect hardest, because a composition that can create a role without it can create a role that outgrows the fence.

**The hand-off will go stale if it is not part of the network change.** `EnvironmentConfig` is a committed copy of Terraform outputs. Crossplane cannot refresh it, by design. A subnet change that lands in the stack and not in `addons/crossplane-platform/values/<env>.yaml` fails quietly, on the next resource someone creates. Treat that values file as a production input. Review it in the same change as the stack when the outputs move.

**Defaults belong to the caller people actually use.** The SNS raw-delivery flag is the concrete case. Schema default `true`, chart default `false`, and a DLQ full of envelopes the consumer could not parse. The chart always sets the field, so the schema default is dead code on the supported path. Dead defaults still apply to the unsupported path. Either make the chart the only supported writer, or make the two defaults match.

**Helm and Go templates share a delimiter.** Escape the composition template, or Helm will resolve it during the addon render. The symptom appears later, on a claim, which is a long way from the line that was added to the chart.

**Order the first sync on purpose.** Application waves for the addon set, resource waves inside the platform chart, server-side apply where the objects accept it, and skip-dry-run where they do not. A green second sync is not evidence the waves are right.

**One cluster is one failure domain.** The `platform` taint keeps ordinary workloads off the nodes where Argo CD and Crossplane run. The `workloads` namespace label gives network policy a handle. An EKS upgrade, a broken CNI change, or a cluster-scoped misconfiguration still takes the reconciler down together with the services it reconciles. Node pools separate noisy neighbors. They do not separate availability.

**Duplicated conventions drift.** "Topic first, subscription owned by the queue" is implemented in the Terraform modules and again in the XRD and the composition. The name prefix is implemented in the IAM policy and again in the Go template. Nothing generates one from the other. A sentence in the XRD description that states the rule is cheap and is the thing a later edit is likely to see. When the Terraform module gains a fourth required tag, the composition does not learn about it until someone notices the drift in AWS.

## A management cluster is the next step

The in-cluster secret carries `tier: application`, so addon discovery and app discovery both deploy onto the cluster where Argo CD is running. That is the current line: one environment, one cluster, with the account, ownership, and node-pool splits above.

The generators are already a fan-out. They keep clusters whose label matches, and each Application's destination is that cluster's server. A dedicated control-plane cluster is another registration. Run Argo CD there, leave `tier: application` off its cluster secret, and set that label only on the workload clusters. The same two ApplicationSets then target those clusters, and the management cluster stays out of the matrix.

Two details follow from the generators as they are written. The Crossplane addon is selected by the same label, so this change installs providers, XRDs, and the EnvironmentConfig on the workload clusters. Leaving Crossplane itself on the management cluster means giving that addon a different selector. And each workload cluster still needs its own `aws-network` EnvironmentConfig, because account, VPC, subnets, and OIDC issuer differ, with the IAM fence repeated per workload account.

What the extra cluster buys is a failure domain the node taint cannot provide. Upgrading a workload cluster leaves the Argo CD that deploys to it untouched. Accounts, Terraform-versus-Crossplane ownership, the EnvironmentConfig copy, the two Helm charts, and the sync waves stay as they are. The hub is an additional line the selectors already allow. The environments running today draw the three lines inside a single cluster.
{% endraw %}
