---
title: "Terraform, Helm, and Crossplane: A Pattern for the Control Plane and the Data Plane"
collection: publications
permalink: /publications/crossplane-helm-terraform
excerpt: 'A pattern I use with companies that want Terraform for the foundation and Crossplane, packaged with Helm, for per-app infrastructure. Argo CD onboards any repository that carries a topic and any cluster that carries a label. The control plane and the data plane share one cluster per environment, split by account, ownership, and node pool.'
date: 2026-10-31
venue: 'Personal website'
tags:
  - crossplane
  - terraform
  - helm
  - argocd
  - gitops
---
{% raw %}
## Summary

When a company wants developers to create their own cloud resources, the pattern I reach for has three tools and a strict split of ownership. It is the pattern I have helped teams adopt.

**Terraform** owns the foundation: the network, the cluster, the IAM boundaries, and the first install of Argo CD. **Crossplane** owns the cloud resources that belong to one service, and I package that API as Helm charts so a service author declares a queue the same way they declare a container port. **Argo CD** is the loop that delivers both. It discovers clusters by a label and repositories by a topic, so onboarding is a metadata change, not a new manifest copied by hand.

I keep the control plane and the data plane on the same cluster in each environment. The split is logical. Accounts separate shared state from workloads. Ownership separates Terraform from Crossplane. A tainted node pool separates platform pods from application pods. A dedicated management cluster, with Argo CD aimed at many workload clusters, is a later step. The ApplicationSets below are already written as that fan-out.

The examples use placeholder names, account ids, and network ids. Copy the shape. Replace the names with your own.

## What you get when the split is real

These are the outcomes I look for when a team adopts the pattern. The rest of the article is how each one is implemented, and where it bites.

- **Developer self-service.** A service team adds a bucket or a queue in the repository they already own. They do not open a platform ticket, and they do not wait for a Terraform plan, for a resource the platform has already published as a claim.
- **Guardrails and least privilege.** Crossplane's AWS role is written in Terraform. It can create resources under a name prefix, it must attach a permissions boundary to every role it creates, and it is denied the Terraform state bucket and KMS key destruction. The claim author chooses `uploads`. The composition chooses the account, the region, and the full cloud name.
- **Clear ownership.** If replacing the object can take the platform down, Terraform owns it. If the object belongs to one service and changes on that service's schedule, Crossplane owns it. Helm is the package format on both sides of that line.
- **Consistency across environments.** Staging and production run the same modules, the same charts, and the same compositions. What changes is an `EnvironmentConfig`: account, VPC, subnets, OIDC issuer. A queue named `orders` becomes `app-orders-staging` in one account and `app-orders-production` in another, from the same values file.
- **Less ticket-ops.** A new service is a GitHub topic. A new cluster is a label. A new queue is a few lines of YAML. The tickets that remain are the ones that should remain: a new kind of cloud resource, or a change to the network and the cluster.
- **Auditability.** The IAM policy, the composition, the claim, and the Application are all in git. Terraform's plan and Argo CD's sync history are the record of what changed. A resource tagged `managed-by: crossplane` is distinguishable from one Terraform still owns. The list of onboarded services is the list of repositories carrying the topic, which GitHub already audits.

## One cluster, three lines

Each environment gets one Kubernetes cluster. Argo CD and Crossplane run on it, and the workloads run on it too. I separate the two planes with three lines, and I am explicit about that with the teams I work with, because "control plane" is a phrase people use for a hub cluster they do not have yet.

**Accounts.** A shared account holds the Terraform state bucket and the credentials the pull-request pipeline uses to apply it. Each workload account holds the VPC, the cluster, and the per-app cloud resources for one environment. State is central. I key it by the stack path and I prefer the S3 lockfile over a separate lock table. Crossplane's IAM policy denies access to that bucket, so the reconciler cannot read or rewrite the foundation's state.

**Ownership.** Terraform owns the long-lived objects: network, cluster, platform KMS, the secret containers, the Crossplane IRSA role, the permissions boundary, and Argo CD itself. Crossplane owns the short-lived, per-app objects: S3 buckets, SQS queues with their DLQ and SNS subscriptions, SNS topics, KMS keys, Secrets Manager secrets, and application IRSA roles. Where both layers create the same kind of object, they follow one naming rule and one dependency rule. A topic exists before a queue subscribes to it, and the subscription is owned by the queue.

**Nodes.** A node pool named `platform` carries the taint `workload=platform:NoSchedule`. Argo CD, Crossplane, and the cluster addons tolerate it and set a matching node selector. Application pods land on the other pools, whatever mix of on-demand, spot, and specialized capacity the company needs. Application namespaces get a `workloads=true` label so network policy has a handle.

```text
shared account                         workload account (one per environment)
  Terraform state bucket                 Terraform
  pipeline role for plans/applies          VPC, cluster, platform KMS
        ^                                  secret containers
        |                                  Crossplane IRSA role
        | state, keyed by stack path       app permissions boundary
        |                                  Argo CD, installed once
        |
        +-------------------------------- one cluster
                                           node pool "platform"
                                             taint workload=platform:NoSchedule
                                             Argo CD, Crossplane, addons
                                           workload node pools
                                             service pods
                                             Crossplane claims
                                               S3, SQS, SNS, KMS, secrets, IRSA
```

A developer can add a queue by editing values. They cannot replace the VPC by editing a claim. A bad chart sync can still affect the cluster both planes share. That shared fate is the trade-off I accept until a management cluster is worth the extra account and the extra operational surface. The label selector is what makes that later step a registration change.

## Terraform owns the foundation

I split Terraform into two repositories, because the reasons to version them are different.

The **module catalog** is reusable: VPC, EKS, IAM role, the small module that installs Argo CD. It is released by tag. The **live stacks** pin that tag and pass the inputs for each environment. Staging and production call the same module. They differ in CIDRs, account ids, and cluster names, and in nothing else I can avoid.

```hcl
terraform {
  source = "git::https://github.com/example/terraform-modules.git//modules/aws/eks?ref=${local.module_version}"
}

dependency "vpc" {
  config_path = "../network"
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

`plan` works before the VPC exists. `apply` still requires the real outputs. The allow-list is what keeps a mocked id out of a real apply. I have seen a plan that looked green because the mock was accepted on apply. Limit the commands.

Remote state lives in the shared account:

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
}
```

Plans and applies run from a pull request. After the cluster exists, Terraform has one more job, and then it should get out of the way: install Argo CD and register the cluster with the labels the ApplicationSets match on.

```hcl
resource "helm_release" "argocd" {
  name       = "argocd"
  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argo-cd"
  namespace  = "argocd"

  values = [yamlencode({
    extraObjects = [{
      apiVersion = "argoproj.io/v1alpha1"
      kind       = "Application"
      metadata = {
        name      = "bootstrap"
        namespace = "argocd"
        annotations = {
          "helm.sh/hook" = "post-install,post-upgrade"
        }
      }
      spec = {
        project = "default"
        source = {
          repoURL        = "https://github.com/example/gitops.git"
          targetRevision = "main"
          path           = "bootstrap"
          directory      = { include = "*.yaml" }
        }
        destination = {
          server    = "https://kubernetes.default.svc"
          namespace = "argocd"
        }
        syncPolicy = {
          automated = { prune = true, selfHeal = true }
        }
      }
    }]
  })]
}
```

That bootstrap Application is a Helm hook. It syncs the GitOps directory that contains the ApplicationSets, and from then on Argo CD owns addons and apps. I treat the Helm release as foundation and I change it rarely. Deleting it deletes the bootstrap.

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

The same stack writes the cluster secret labels: `tier=application`, the environment name, and the account id. Those labels are the contract for the discovery section below. Repository credentials come from the secret store. They do not go in git.

## The IAM role is the guardrail

Self-service without a fence is a shared admin role with extra steps. The fence is an IRSA role Terraform creates, one per environment, assumed by the Crossplane provider service account. The Helm chart only receives the role ARN. It annotates the service account through a `DeploymentRuntimeConfig`, and the `ProviderConfig` says `credentials.source: IRSA`.

```hcl
policy_json = jsonencode({
  Version = "2012-10-17"
  Statement = [
    {
      Sid      = "IamRolesRequireBoundary"
      Effect   = "Allow"
      Action   = ["iam:CreateRole", "iam:PutRolePolicy", "iam:AttachRolePolicy"]
      Resource = ["arn:aws:iam::<ACCOUNT_ID>:role/app-*"]
      Condition = {
        StringEquals = {
          "iam:PermissionsBoundary" = "arn:aws:iam::<ACCOUNT_ID>:policy/app-boundary-<ENV>"
        }
      }
    },
    {
      Sid      = "S3UnderPrefix"
      Effect   = "Allow"
      Action   = "s3:*"
      Resource = ["arn:aws:s3:::app-*", "arn:aws:s3:::app-*/*"]
    },
    {
      Sid    = "SqsUnderPrefix"
      Effect = "Allow"
      Action = [
        "sqs:CreateQueue",
        "sqs:DeleteQueue",
        "sqs:SetQueueAttributes",
        "sqs:GetQueueAttributes",
        "sqs:TagQueue",
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
      Resource = [
        "arn:aws:s3:::<TFSTATE_BUCKET>",
        "arn:aws:s3:::<TFSTATE_BUCKET>/*",
      ]
    },
  ]
})
```

I want teams to read this policy as the real control plane.

Roles Crossplane creates must match `app-*` and must carry the permissions boundary. The boundary policy is Terraform's, so a claim cannot widen what an application role is allowed to become. Forget the boundary in a Composition and IAM denies the create. That deny is the outcome I want.

SQS is limited by verb and by prefix. S3 in this example is limited by prefix only (`s3:*` on `app-*`). I leave that difference visible on purpose. A prefix stops Crossplane from touching the state bucket and from creating `prod-billing-archive` outside the pattern. It does not stop a composition from reading a neighboring service's bucket when that bucket is also named `app-*`. If two services must be isolated from each other, the policy needs a tighter resource, or the composition must be the only writer and the names must be unique. Say which one you chose. A review that only checks "it is least privilege" will miss it.

Two denies sit on top of the allows. Crossplane cannot schedule deletion of a KMS key, and it cannot touch the state bucket. A bug in a composition, or a leaked set of credentials for this role, stops at the foundation. The account split becomes real at IAM, which is the only place it is real.

The compositions use the same prefix. The author of the claim passes `orders`. The template builds `app-orders-staging`. Account and region come from the environment, not from the pull request.

## Helm packages the platform API and the app

I use two charts, published as OCI artifacts and pulled by Argo CD.

The **platform chart** is the API. It renders one provider package per AWS service, the `ProviderConfig`, an `EnvironmentConfig`, and an XRD plus a Composition for each resource kind the company is willing to offer: IRSA, KMS, S3, Secrets Manager, SNS, SQS. I pin the upstream Crossplane chart to an exact version. I let the chart we publish float on a patch range such as `0.4.*`. A patch rolls forward on the next sync, to every cluster the generator matches. That is fast, and a bad publish is an incident at the same speed. The upstream control-plane chart stays exact for that reason.

The **app chart** is what service authors render. One values file describes the workload (image, ports, probes, resources, routes). Another describes cloud resources under `infrastructure:`. The chart emits the Deployment, Service, HPA, PDB, routes, ExternalSecret, and ServiceAccount, and it emits one claim per entry. A small library chart holds the IRSA fragment so every resource template does not reinvent the role.

Compositions need the account, the region, the VPC, the private subnets, the OIDC issuer, and the boundary ARN. Those values do not belong in a claim, and I do not hardcode them in the Composition. Crossplane is denied the state bucket, so it cannot look them up either. The hand-off is a values file in the GitOps repository, filled from Terraform outputs and rendered into an `EnvironmentConfig`:

```yaml
# gitops/addons/crossplane-platform/values/staging.yaml
aws:
  environment: staging
  region: us-east-1
  accountId: "<ACCOUNT_ID>"
  provider:
    iamRoleArn: "arn:aws:iam::<ACCOUNT_ID>:role/app-crossplane-staging"
  network:
    vpcId: <VPC_ID>
    privateSubnetIds: [<SUBNET_A>, <SUBNET_B>, <SUBNET_C>]
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

`required` fails the render when an environment was not filled in. A failed Helm release is the outcome I want. A queue created in an empty region and marked ready is a much worse one. Every composition starts by loading `aws-network` by name, then reads it from the function context:

```yaml
- step: environment
  functionRef:
    name: function-environment-configs
  input:
    apiVersion: environmentconfigs.fn.crossplane.io/v1beta1
    kind: Input
    spec:
      environmentConfigs:
        - type: Reference
          ref:
            name: aws-network
- step: render
  functionRef:
    name: function-go-templating
- step: ready
  functionRef:
    name: function-auto-ready
```

This file is a copy. It changes when someone updates it. A subnet change is a Terraform pull request and an edit to this values file. Skip the second edit and the cluster stays healthy, existing resources stay where they are, and the next claim is built against a network the VPC stack no longer has. I put that values file in the same review as the stack when outputs move. The deny on the state bucket is why the copy exists. The copy is what can go stale.

### What a service author writes

This is the whole data-plane interface for a bucket:

```yaml
# deploy/values.yaml
image:
  repository: ghcr.io/example/files
  tag: "1.4.2"
ports:
  - name: http
    containerPort: 8080
resources:
  requests:
    cpu: 100m
    memory: 128Mi
---
# deploy/resources.yaml
infrastructure:
  s3:
    buckets:
      - name: uploads
        versioning: true
  sqs:
    queues:
      - name: orders
        visibilityTimeoutSeconds: 30
        dlq:
          maxReceiveCount: 5
        subscriptions:
          - topic: order-events
```

The chart fails the render if `name` is missing. The claim's Kubernetes name is the service name plus the short name, truncated to 63 characters. The AWS name is a different string, built in the composition, so the author cannot escape the prefix:

```text
{{`{{- $name := $params.name | default $claimName }}`}}
{{`{{- $queueName := printf "app-%s-%s" $name $env.environment }}`}}
{{`{{- $dlqName := printf "app-%s-dlq-%s" $name $env.environment }}`}}
```

The composition emits the queue, the DLQ when requested, one SNS subscription per entry, and a queue policy that allows `sns.amazonaws.com` to send from those topic ARNs. The policy selects its queue with `matchControllerRef` and a `role: main` label, so it binds to the queue this claim owns. Subscriptions live on the queue, which keeps the dependency one-directional and matches the rule in the Terraform modules. The XRD says so in its schema description. That sentence is the documentation a later edit is most likely to see.

Shipping the composition inside Helm has a sharp edge. The composition body is a Go template, evaluated later by `function-go-templating`. Helm evaluates `{{ }}` at render time. Escape every Go expression so Helm stores it as text:

```text
# in the chart
{{`{{- $env := index .context "apiextensions.crossplane.io/environment" }}`}}

# stored on the Composition after Helm renders
{{- $env := index .context "apiextensions.crossplane.io/environment" }}
```

The first time a line ships without the escape, the addon sync is green. The claim renders with a blank name, or the function throws. The failure shows up far from the line that caused it.

I also treat chart defaults and schema defaults as two APIs. A case I keep seeing: the XRD defaults SNS `rawMessageDelivery` to `true`, and the app chart defaults it to `false` and always writes the field. Raw delivery hands the subscriber the bare payload. If the consumer parses the SNS envelope, every message fails and lands on the DLQ, while the subscription looks healthy. Because the chart sets the field, the schema default never applies to the path service authors use. A claim written by hand still gets `true`. Pick the caller you support, make that default the one you test, and make the other one match or document it.

### Order the first sync

The first apply fails if providers, the provider config, and the XRDs land in one wave. Argo CD reports a comparison error, someone clicks Sync, and the second attempt works. I do not want a platform that only converges on the second click.

Waves on an Application order Applications. Waves on a resource order objects inside one Application. They are different counters.

| What | Wave | What it waits for |
|---|---|---|
| Application: upstream Crossplane chart | 1 | the CRDs and core controllers this chart installs |
| Application: platform chart (XRDs, Compositions) | 3 | the core chart |
| Provider packages | 1, inside the platform Application | later waves in the same Application |
| ProviderConfig and EnvironmentConfig | 5 | the provider packages |
| XRDs and Compositions | 10 | the environment object and the functions |

Claims arrive with the service Application, after the XRD exists. I set `SkipDryRunOnMissingResource` on provider objects, and I turn server-side apply off for the objects that reject it. Copy the wave numbers and forget those annotations, and the first sync is red again.

## Argo CD discovers work by label

This is the part I insist on, because it is where the ticket actually disappears.

Argo CD ApplicationSets generate Applications from some source of truth outside the Application manifest. I use two generators, usually as a matrix. The **cluster generator** selects clusters by label. The **SCM provider generator** selects Git repositories by a topic. Each pair becomes one Application. Nobody maintains a list of services, and nobody copies an Application YAML when a new environment appears.

### Clusters: a label decides who receives work

Argo CD stores each destination as a Secret. The labels on that Secret are the selector. Terraform writes them when it registers the cluster, which keeps the account id next to the cluster instead of pasted into every service repo.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: staging
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
    tier: application
    cluster: staging
    accountId: "<ACCOUNT_ID>"
type: Opaque
stringData:
  name: staging
  server: https://<CLUSTER_API>
  config: "{}"
```

The generator keeps the secrets whose labels match:

```yaml
generators:
  - clusters:
      selector:
        matchLabels:
          tier: application
```

`tier: application` means "this cluster should receive addons and services." The template reads the other labels. `cluster` becomes part of the Application name and selects the per-environment values file. `accountId` is injected into Helm so the chart can build IRSA ARNs. The service author never types an account id.

A cluster without that label is invisible to the generator. That is how a future management cluster runs Argo CD and stays out of the matrix. Today I put the label on the same cluster Argo CD runs on, which is why one environment is one cluster. Adding a second workload cluster is registering another Secret with `tier: application`. Every addon and every onboarded service is created there on the next refresh, aimed at that cluster's server, with that cluster's account id. Consistency across environments comes from the generator, not from a checklist.

### Repositories: a topic decides who is a service

The SCM provider generator asks GitHub (or GitLab, or another provider) for repositories and filters them. On GitHub, `labelMatch` compares against repository **topics**. I use the topic `argocd-app`.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: app-discovery
  namespace: argocd
spec:
  goTemplate: true
  generators:
    - matrix:
        generators:
          - clusters:
              selector:
                matchLabels:
                  tier: application
          - scmProvider:
              github:
                organization: example
                # GitHub App credentials live in a cluster Secret, not in git.
                appSecretName: github-app
              filters:
                - repositoryMatch: ".*"
                  labelMatch: argocd-app
  template:
    metadata:
      name: '{{ .repository }}-{{ index .metadata.labels "cluster" }}'
    spec:
      project: apps
      sources:
        - repoURL: ghcr.io/example/charts
          chart: apps
          targetRevision: "0.4.*"
          helm:
            ignoreMissingValueFiles: true
            valueFiles:
              - "$values/deploy/values.yaml"
              - "$values/deploy/resources.yaml"
              - '$values/deploy/values/{{ index .metadata.labels "cluster" }}.yaml'
              - '$values/deploy/resources/{{ index .metadata.labels "cluster" }}.yaml'
            values: |
              environment: {{ index .metadata.labels "cluster" }}
              serviceAccount:
                aws:
                  accountId: {{ index .metadata.labels "accountId" | quote }}
        - repoURL: "{{ .url }}"
          targetRevision: "{{ .branch }}"
          ref: values
      destination:
        server: "{{ .server }}"
        namespace: "{{ .repository }}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
        managedNamespaceMetadata:
          labels:
            workloads: "true"
```

Read the matrix as a sentence. For every cluster labeled `tier=application`, and for every repository in the `example` organization whose topics include `argocd-app`, render one Application. The chart comes from the OCI registry. The values come from that repository's `deploy/` directory. Missing per-environment files are ignored, so a service with only `deploy/values.yaml` and `deploy/resources.yaml` is valid. The destination namespace is the repository name, labeled `workloads=true`.

`goTemplate: true` matters. The `index` calls and the injected `values:` block are Go templates, evaluated by the ApplicationSet controller when it stamps each Application. They are not Helm yet. Helm runs later, inside the chart.

Addon discovery is the same cluster selector crossed with a git file generator, one small config file per addon directory, instead of the SCM provider. I keep addons in the platform repository and services in their own repositories. The cluster half of the matrix is identical, which is what makes a new environment pick up both.

```yaml
# The addon half of the matrix. Same cluster selector, different second generator.
generators:
  - matrix:
      generators:
        - clusters:
            selector:
              matchLabels:
                tier: application
        - git:
            repoURL: https://github.com/example/gitops.git
            revision: main
            files:
              - path: "addons/*/config.yaml"
```

A config file is enough to add an addon. `wave` becomes the Application sync wave, so Crossplane's core chart lands before the chart that publishes XRDs.

```yaml
addonDir: crossplane
chartRepo: https://charts.crossplane.io/stable
chartName: crossplane
chartVersion: "<exact-version>"   # pin it; the control-plane chart stays exact
namespace: crossplane-system
wave: "1"
```

Service Applications go to an AppProject named `apps`. Addon Applications go to a project named `addons`. That split is where I constrain a service repository to workloads and claims, and leave CRDs, providers, and cluster-scoped policy to the platform repository. The topic is a wide door. The AppProject is the frame around it.

### Why the label is the product

I have watched teams keep a folder of Application manifests, one per service per environment, updated by whoever remembered. It works at five services. It rots at thirty. The generators replace that folder with two questions GitHub and Kubernetes can already answer.

**Onboarding a service is a topic.** The team creates the repository, adds `deploy/values.yaml`, and sets the topic `argocd-app`. The next ApplicationSet refresh creates `billing-staging` and `billing-production` if both clusters carry `tier=application`. There is no platform ticket to "add my app to Argo," and there is no YAML to copy and rename. Offboarding is removing the topic. With automated prune, the Applications go away on the same loop.

**Onboarding an environment is a label.** Register the cluster Secret with `tier: application`, `cluster: production`, and the production account id. The addon matrix and the app matrix both fire. Every current service is deployed there with the production values file when it exists, and with the production account id injected either way. You do not edit forty repositories to tell them a new cluster exists.

**The account id stays off the developer's keyboard.** Injecting `accountId` from the cluster label is a small line in the template and it removes a class of mistakes I see constantly: staging deploys with a production account id copied from a README. The EnvironmentConfig does the same job for VPC and OIDC on the Crossplane side. Developers describe the resource. The platform describes the place.

**Audit is the metadata plus git.** Who is in production is the set of repositories with the topic, which GitHub records, crossed with the clusters carrying the label, which is a Secret in git or a Terraform resource. What they deployed is the chart version and the values in the service pull request. What they were allowed to create in AWS is the IAM policy and the composition. I can answer "who added a queue" without CloudTrail archaeology: it is the pull request that edited `deploy/resources.yaml`.

**Drift correction is on by default.** `selfHeal` and `prune` mean a manual edit in the cluster comes back to git, and a claim removed from values is deleted. That is the audit property people forget to ask for. A console change is not a durable change.

The power has a cost, and I put it in the open.

Anyone who can set a topic can onboard a repository. Decide who that is. Branch protection does not cover topics, so treat topic changes as a privileged action, or restrict them with organization rules.

`repositoryMatch: ".*"` together with `labelMatch` is intentional. Drop the topic filter and the generator onboards the entire organization, including repositories that happen to contain a `deploy/` folder from a tutorial. I have seen that happen in a proof of concept that was left running.

`ignoreMissingValueFiles: true` lets a repository onboard before every environment file exists. The sync then fails in the open until `deploy/values.yaml` is valid. That failure is the signal. A green Application with an empty release is worse.

The matrix multiplies. Forty services and three clusters is 120 Applications, which Argo CD handles comfortably. Four hundred services and ten clusters is a capacity conversation: controller memory, refresh interval, and how often the SCM provider lists the organization. I would rather have that conversation at design time than discover it as a stuck queue of reconciliations.

A topic is also a coarse switch. It says "this repository is a service," not "this repository may deploy to production." When a company needs that second decision, I add a second topic or a path the filter requires, and I keep production as its own cluster label. I do not fork the ApplicationSet per team. The moment each team has a copy, the generator has stopped being the source of truth.

## Trade-offs

I recommend this pattern with the costs attached. Teams that adopt it and pretend the costs are not there are the ones who call me back.

**Two tools means a hand-off.** Terraform will not see claims. Crossplane will not see the state file. The `EnvironmentConfig` is a copy, and copies go stale when a network change updates only one side. Make the values file part of the network review.

**The IAM fence is uneven if you let it be.** A prefix is a guardrail. It becomes isolation between services only when you design it that way. Verb-scoped SQS and prefix-scoped S3 are different promises. Write down which promise each new composition makes.

**Self-service starts after the API exists.** A queue the chart already knows is a values change. A new AWS product is a module, an XRD, a composition, an IAM statement, and a chart template. That lead time is the ownership boundary doing its job. If every request still needs a new composition, the platform API is too small, or the team is using Crossplane as a slower Terraform.

**Defaults drift.** Schema defaults, chart defaults, and Terraform module defaults are three places to express "a queue has a DLQ with five receives." They will disagree. The caller service authors actually use is the chart. Test that path.

**Helm and Go templates share a delimiter.** Escape composition templates inside the chart. The symptom appears on the claim.

**One cluster is one failure domain.** The platform taint keeps a noisy build off the Crossplane nodes. It does not keep an upgrade, a CNI change, or a cluster-scoped network policy from taking Argo CD down together with the services. I accept that while the environment is small enough that a second cluster is more risk than it removes.

**Conventions are duplicated.** "Topic first, subscription on the queue" lives in the Terraform module and again in the XRD. The name prefix lives in the IAM policy and again in the Go template. Nothing generates one from the other. A sentence in the XRD is cheap insurance. When the module gains a required tag, the composition does not learn it until someone notices two styles of resource in the console.

**Discovery is only as strict as the topic and the AppProject.** The generator will faithfully deploy a repository that should not have been labeled. Spend the time on who can apply `argocd-app`, and on what the `apps` project is allowed to create.

## A management cluster is a later step

What I have described is one cluster per environment. Argo CD, Crossplane, and the workloads share it. The `tier: application` label is on that cluster's Secret, so the generators deploy onto the place they run.

The generators are already a fan-out. They select clusters by label, and each Application's destination is that cluster's server. When a company is ready for a hub, the change is a registration. Run Argo CD on a management cluster, leave `tier: application` off its Secret, and set that label only on the workload clusters. The same ApplicationSets target those clusters. The management cluster stays out of the matrix.

Two details follow from the generators as written. The Crossplane addon uses the same cluster selector, so this change installs providers and XRDs on the workload clusters. Keeping Crossplane on the management cluster means a different selector for that addon. And each workload cluster still needs its own `EnvironmentConfig`, because account, VPC, and OIDC issuer differ. The IAM role is still created per workload account, by Terraform.

What the extra cluster buys is a failure domain the node taint cannot provide. Upgrading a workload cluster leaves the Argo CD that deploys to it running. What it does not replace is the logical split: accounts, Terraform for the foundation, Crossplane for the app, Helm as the package, and discovery by label. Those are the lines I want in place first. The hub is an additional line, and the selector is already shaped for it.
{% endraw %}
