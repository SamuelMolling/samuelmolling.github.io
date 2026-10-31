---
title: "Crossplane, Helm, and Argo CD: A Pattern for the Control Plane and the Data Plane"
collection: publications
permalink: /publications/crossplane-helm-terraform
excerpt: 'A pattern I use when a company wants per-app cloud resources without a ticket for every queue. Crossplane, packaged as Helm charts, is the API. Argo CD onboards repositories by topic and clusters by label. NetworkPolicy is a short values list the chart turns into default-deny. The cluster and the first Argo CD install are foundation work, and then that layer steps back.'
date: 2026-10-31
venue: 'Personal website'
tags:
  - crossplane
  - helm
  - argocd
  - gitops
---
{% raw %}
## Summary

When a company wants developers to create their own cloud resources, the pattern I reach for is Crossplane for the resources, Helm for the package, and Argo CD for delivery. It is the pattern I have helped teams adopt.

**Crossplane** owns the cloud resources that belong to one service. **Helm** is how I ship that API, and how a service author declares a queue the same way they declare a container port or a network allow. **Argo CD** discovers clusters by a label and repositories by a topic, so onboarding is a metadata change, not a new manifest copied by hand. Terraform, or whatever builds the cluster, provisions the network and the cluster and bootstraps Argo CD. Then it steps back.

I keep the control plane and the data plane on the same cluster in each environment. The split is logical. A tainted node pool separates platform pods from application pods. Ownership separates the foundation from the claims. A dedicated management cluster, with Argo CD aimed at many workload clusters, is a later step. The ApplicationSets below are already written as that fan-out.

The examples use placeholder names, account ids, and network ids. Copy the shape. Replace the names with your own.

## What you get when the split is real

These are the outcomes I look for when a team adopts the pattern. The rest of the article is how each one is implemented, and where it bites.

- **Developer self-service.** A service team adds a bucket, a queue, or a network allow in the repository they already own. For a resource the platform has published as a claim or a chart value, that is the whole change.
- **Guardrails.** The composition builds the cloud name, encryption, and tags. NetworkPolicy defaults to deny. An AppProject limits what a service repository may sync, and admission rejects privileged pods, floating image tags, and claims that try to pick their own composition.
- **Clear ownership.** The foundation owns the network, the cluster, and the Argo CD install. Crossplane owns the objects that belong to one service and change on that service's schedule. Helm is the package on both sides of the platform API: the chart that publishes XRDs, and the chart that emits claims.
- **Consistency across environments.** Staging and production run the same charts and the same compositions. What changes is an `EnvironmentConfig`: account, VPC, subnets, OIDC issuer. A queue named `orders` becomes `app-orders-staging` in one place and `app-orders-production` in another, from the same values file.
- **Less ticket-ops.** A new service is a GitHub topic. A new cluster is a label. A new queue is a few lines of YAML. The tickets that remain are a new kind of cloud resource, or a change to the network and the cluster.
- **Auditability.** The composition, the claim, the NetworkPolicy values, and the Application are in git. Argo CD's sync history is the record of what the cluster did with them. A resource tagged `managed-by: crossplane` is the one a claim created. The list of onboarded services is the list of repositories carrying the topic, which GitHub already audits.

## One cluster, then the foundation steps back

Each environment gets one Kubernetes cluster. Argo CD and Crossplane run on it, and the workloads run on it too. I am explicit about that with the teams I work with, because "control plane" is a phrase people use for a hub cluster they do not have yet.

The logical split is ownership and placement.

**Ownership.** The foundation provisions the network and the cluster, installs Argo CD, and stops there. Crossplane owns the per-app objects: S3 buckets, SQS queues with their DLQ and SNS subscriptions, SNS topics, KMS keys, and secrets. A topic exists before a queue subscribes to it, and the subscription is owned by the queue.

**Nodes.** A node pool named `platform` carries the taint `workload=platform:NoSchedule`. Argo CD, Crossplane, and the cluster addons tolerate it. Application pods land on the other pools. Application namespaces get a `workloads=true` label so network policy has a handle.

```text
foundation (once per environment)
  network, cluster, Argo CD bootstrap
        |
        v
one cluster
  node pool "platform"          workload node pools
    Argo CD, Crossplane, addons   service pods
                                  Crossplane claims (S3, SQS, SNS, KMS, secrets)
                                  NetworkPolicies from the app chart
```

A developer adds a queue by editing values. Replacing the VPC is foundation work, outside the claim. A bad chart sync can still affect the cluster both planes share. That shared fate is the trade-off I accept until a management cluster is worth the extra operational surface. The label selector is what makes that later step a registration change.

## The foundation, then it steps back

Whatever tool builds the account, I use it for three things: the network, the cluster, and the first install of Argo CD. It also registers the cluster Secret with `tier=application`, the environment name, and the account id. Those labels are the contract for discovery below. Repository credentials come from the secret store. They do not go in git.

The install leaves behind one Application, as a post-install hook, aimed at the GitOps repository. After that hook syncs, Argo CD owns addons and apps. I change the bootstrap rarely.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bootstrap
  annotations:
    helm.sh/hook: post-install,post-upgrade
spec:
  source:
    repoURL: https://github.com/example/gitops.git
    path: bootstrap
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

From here a new queue is a values file, a new service is a GitHub topic, and a new cluster is a label.

## Helm packages the platform API and the app

I use two charts, published as OCI artifacts and pulled by Argo CD.

The **platform chart** is the API. It renders one provider package per AWS service, the `ProviderConfig`, an `EnvironmentConfig`, and an XRD plus a Composition for each resource kind the company is willing to offer: KMS, S3, Secrets Manager, SNS, SQS. I pin the upstream Crossplane chart to an exact version. I let the chart we publish float on a patch range such as `0.4.*`. A patch rolls forward on the next sync, to every cluster the generator matches. That is fast, and a bad publish is an incident at the same speed. The upstream control-plane chart stays exact for that reason.

The **app chart** is what service authors render. One values file describes the workload (image, ports, probes, resources, routes, and who the process may talk to). Another describes cloud resources under `infrastructure:`. The chart emits the Deployment, Service, HPA, PDB, routes, ExternalSecret, ServiceAccount, and the NetworkPolicies described below, and it emits one claim per entry.

Compositions need the account, the region, the VPC, the private subnets, and the OIDC issuer. Those values do not belong in a claim, and I do not hardcode them in the Composition. The hand-off is a values file in the GitOps repository, filled when the foundation changes, and rendered into an `EnvironmentConfig`:

```yaml
# gitops/addons/crossplane-platform/values/staging.yaml
aws:
  environment: staging
  region: us-east-1
  accountId: "<ACCOUNT_ID>"
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

This file is a copy of the place the cluster lives. It changes when someone updates it. A subnet change in the foundation has to land here too. Skip that edit and the cluster stays healthy, existing resources stay where they are, and the next claim is built against a network the cluster no longer has.

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

The composition emits the queue, the DLQ when requested, one SNS subscription per entry, and a queue policy that allows `sns.amazonaws.com` to send from those topic ARNs. The policy selects its queue with `matchControllerRef` and a `role: main` label, so it binds to the queue this claim owns. Subscriptions live on the queue, which keeps the dependency one-directional: the topic exists first. The XRD says so in its schema description. That sentence is the documentation a later edit is most likely to see.

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

Waves order creation. Health decides when Argo CD is allowed to call the Application healthy. A claim whose AWS resource is still creating must stay `Progressing`. I map Crossplane's `Ready=True` condition into Argo CD's health check, so a green Application means the queue exists, not only that the YAML was applied.

## NetworkPolicy the developer never writes

The same app chart is what makes network policy feel small. A service author lists who may connect and where the process may connect. The template turns that list into a default-deny policy plus explicit allows, in the app namespace, on every environment the ApplicationSet targets.

This is the whole interface I want them to see:

```yaml
# deploy/values.yaml
networkPolicy:
  ingress:
    fromNamespaces:
      - gateway          # the shared ingress namespace
    fromApps:
      - checkout         # another app namespace, labeled workloads=true
  egress:
    allowDNS: true
    toApps:
      - name: payments
        ports: [8080]
```

`fromNamespaces` is a platform namespace that is allowed to call in, typically the gateway. `fromApps` is another service onboarded the same way, so its namespace carries `workloads=true` and is named after the repository. `allowDNS` is the baseline almost every pod needs. `toApps` is egress to a specific service, with the ports the caller actually uses. There is no `podSelector`, no `policyTypes`, and no `ipBlock` in the values file.

With `networkPolicy` left at the chart defaults, the release still renders a default-deny. DNS is allowed unless the values set `allowDNS: false`. Same-namespace traffic is allowed, because the pods of one service have to reach each other. Everything else waits for a line in this file.

For a service named `billing`, the chart renders two objects in the `billing` namespace.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: billing-default-deny
  namespace: billing
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: billing-allow
  namespace: billing
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: gateway
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: checkout
              workloads: "true"
  egress:
    # pods of this service, same namespace
    - to:
        - podSelector: {}
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - { protocol: UDP, port: 53 }
        - { protocol: TCP, port: 53 }
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: payments
              workloads: "true"
      ports:
        - { protocol: TCP, port: 8080 }
```

`podSelector: {}` means every pod in that namespace. That is safe here because the namespace belongs to one Application. The ApplicationSet creates it and labels it `workloads=true`. An empty `podSelector` on the deny policy selects all of them. Kubernetes unions policies, so the allow policy adds flows on top of the deny. An empty `ingress` list denies ingress. An allow-all is the different shape, `ingress: [{}]`, and the template never emits it.

The template that produces the allow rules stays in the chart. A shortened form:

```yaml
{{- range .Values.networkPolicy.ingress.fromNamespaces }}
- from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: {{ . | quote }}
{{- end }}
{{- range .Values.networkPolicy.ingress.fromApps }}
- from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: {{ . | quote }}
          workloads: "true"
{{- end }}
{{- if .Values.networkPolicy.egress.allowDNS | default true }}
- to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
  ports:
    - { protocol: UDP, port: 53 }
    - { protocol: TCP, port: 53 }
{{- end }}
```

The platform team owns that template. Service authors own the list of names.

A few properties are why I put it in the shared chart.

**Secure by default.** Every app namespace ships with default-deny. A new service is closed until someone lists a caller and a callee. Turning that off is a values change a reviewer can see. Forgetting an entry fails closed: the pod times out, and the fix is a line in git.

**Consistent across apps and environments.** Staging and production render the same policies from the same template. A per-environment values file can add a caller that exists only in production. The shape of the policy does not change between clusters, and it does not change between the billing service and the payments service.

**Reviewed in git.** The allow list is a pull request on `deploy/values.yaml`. "Who can call billing, and what billing can call" is a diff, not a `kubectl` session. Argo CD's prune removes an allow when the line is removed.

**Authors skip the NetworkPolicy syntax.** Most service developers should not have to remember that a missing `namespaceSelector` means "this namespace," or that an empty `from` allows the world. They name a namespace or an app and, for egress, a port. The chart fills in selectors, DNS, and the deny.

**The platform evolves the policy in one place.** When the DNS pods move labels, or when every service should also reach the metrics scraper, that change is a chart release. Argo CD rolls it to every app on the next sync. Nobody edits forty copies of a NetworkPolicy.

Two limits belong in the open, because they are how NetworkPolicy actually works.

The policy is namespace-scoped. The chart can allow billing to *receive* traffic from checkout, and it can allow billing to *open* traffic to payments. Checkout's own chart still has to allow that egress, and payments' chart still has to allow that ingress. An egress rule on the caller does not open the callee. I tell teams to update both values files in the same change, and to expect a timeout when only one side moved.

The objects are inert until the cluster CNI enforces NetworkPolicy. That switch is foundation work, done when the cluster is built. Developers inherit enforcement. They do not turn it on per app.

## Argo CD discovers work by label

This is the part I insist on, because it is where the ticket actually disappears.

Argo CD ApplicationSets generate Applications from some source of truth outside the Application manifest. I use two generators, usually as a matrix. The **cluster generator** selects clusters by label. The **SCM provider generator** selects Git repositories by a topic. Each pair becomes one Application. Nobody maintains a list of services, and nobody copies an Application YAML when a new environment appears.

### Clusters: a label decides who receives work

Argo CD stores each destination as a Secret. The labels on that Secret are the selector. The foundation writes them when it registers the cluster, which keeps the account id next to the cluster instead of pasted into every service repo.

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

`tier: application` means "this cluster should receive addons and services." The template reads the other labels. `cluster` becomes part of the Application name and selects the per-environment values file. `accountId` is injected into Helm so claims target the right account. The service author never types an account id.

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

Service Applications go to an AppProject named `apps`. Addon Applications go to a project named `addons`. The topic is a wide door. The project, and the admission rules next to it, are the frame. Both are below.

### Why the label is the product

I have watched teams keep a folder of Application manifests, one per service per environment, updated by whoever remembered. It works at five services. It rots at thirty. The generators replace that folder with two questions GitHub and Kubernetes can already answer.

**Onboarding a service is a topic.** The team creates the repository, adds `deploy/values.yaml`, and sets the topic `argocd-app`. The next ApplicationSet refresh creates `billing-staging` and `billing-production` if both clusters carry `tier=application`. There is no platform ticket to "add my app to Argo," and there is no YAML to copy and rename. Offboarding is removing the topic. With automated prune, the Applications go away on the same loop.

**Onboarding an environment is a label.** Register the cluster Secret with `tier: application`, `cluster: production`, and the production account id. The addon matrix and the app matrix both fire. Every current service is deployed there with the production values file when it exists, and with the production account id injected either way. You do not edit forty repositories to tell them a new cluster exists.

**The account id stays off the developer's keyboard.** Injecting `accountId` from the cluster label is a small line in the template and it removes a class of mistakes I see constantly: staging deploys with a production account id copied from a README. The EnvironmentConfig does the same job for VPC and OIDC on the Crossplane side. Developers describe the resource. The platform describes the place.

**Audit is the metadata plus git.** Who is in production is the set of repositories with the topic, which GitHub records, crossed with the clusters carrying the label. What they deployed is the chart version and the values in the service pull request. What the queue looks like in the cloud is the composition. I can answer "who added a queue" from the pull request that edited `deploy/resources.yaml`.

**Drift correction is on by default.** `selfHeal` and `prune` mean a manual edit in the cluster comes back to git, and a claim removed from values is deleted. That is the audit property people forget to ask for. A console change is not a durable change.

The power has a cost, and I put it in the open.

Anyone who can set a topic can onboard a repository. Decide who that is. Branch protection does not cover topics, so treat topic changes as a privileged action, or restrict them with organization rules.

`repositoryMatch: ".*"` together with `labelMatch` is intentional. Drop the topic filter and the generator onboards the entire organization, including repositories that happen to contain a `deploy/` folder from a tutorial. I have seen that happen in a proof of concept that was left running.

`ignoreMissingValueFiles: true` lets a repository onboard before every environment file exists. The sync then fails in the open until `deploy/values.yaml` is valid. That failure is the signal. A green Application with an empty release is worse.

The matrix multiplies. Forty services and three clusters is 120 Applications, which Argo CD handles comfortably. Four hundred services and ten clusters is a capacity conversation: controller memory, refresh interval, and how often the SCM provider lists the organization. I would rather have that conversation at design time than discover it as a stuck queue of reconciliations.

A topic is also a coarse switch. It says "this repository is a service," not "this repository may deploy to production." When a company needs that second decision, I add a second topic or a path the filter requires, and I keep production as its own cluster label. I do not fork the ApplicationSet per team. The moment each team has a copy, the generator has stopped being the source of truth.

## Guardrails around the self-service door

Self-service is a wide door on purpose. The guardrails are what make the width safe. I keep them in the platform charts and in cluster policy, so a service author inherits them and a reviewer sees an exception as a values change.

### AppProject and who may sync

The `apps` project lists the chart registry and the organization as the only sources, allows destinations only on clusters the platform registered, and whitelists the kinds a service is allowed to create. Cluster-scoped kinds stay empty. CRDs, providers, and compositions belong to the `addons` project.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: apps
  namespace: argocd
spec:
  sourceRepos:
    - ghcr.io/example/charts
    - https://github.com/example/*
  destinations:
    - name: "*"
      namespace: "*"
  clusterResourceWhitelist: []
  namespaceResourceWhitelist:
    - { group: "", kind: Service }
    - { group: apps, kind: Deployment }
    - { group: networking.k8s.io, kind: NetworkPolicy }
    - { group: external-secrets.io, kind: ExternalSecret }
    - { group: crossplane.example.com, kind: AWSSQSQueue }
    - { group: crossplane.example.com, kind: AWSS3Bucket }
  roles:
    - name: service
      groups: [app-developers]
      policies:
        - p, proj:apps:service, applications, get, apps/*, allow
        - p, proj:apps:service, applications, sync, apps/*, allow
```

A developer can read and sync the Application the generator created for their repository. Updating the project, adding a kind, or pointing a source at an unknown registry is a platform change. That is the RBAC I want: the topic onboards the repo, and the project decides what the repo is allowed to become.

### Pods, images, and admission

Every app namespace is labeled for [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) at `restricted`, next to `workloads=true`. The ApplicationSet sets both labels when it creates the namespace, so a new service does not get a looser namespace than the last one. `restricted` already rejects privileged containers, root, and host namespaces.

```yaml
labels:
  workloads: "true"
  pod-security.kubernetes.io/enforce: restricted
  pod-security.kubernetes.io/warn: restricted
```

I add a small admission policy on top, with Kyverno or Gatekeeper. Either tool is fine. The rules I actually enforce:

- containers declare CPU and memory limits
- `latest` and other floating tags are rejected; the image is a digest
- the image is signed (cosign) by a key the platform trusts
- a namespace labeled `workloads=true` has a default-deny NetworkPolicy before pods are admitted
- claim kinds are the ones the platform chart publishes, and `compositionRef` is empty so a service cannot select a different composition

The pod rule is the one worth showing. The others are the same policy object with a different `match`.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: app-baseline
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-limits-and-digest
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaceSelector:
                matchLabels:
                  workloads: "true"
      validate:
        message: "set cpu and memory limits, and pin the image by digest"
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    cpu: "?*"
                    memory: "?*"
                image: "*@sha256:*"
```

The values file carries the digest. The tag is for humans, in the pull request description, not in the spec the cluster runs:

```yaml
image:
  repository: ghcr.io/example/files
  digest: sha256:<digest>
```

Admission is also where a missing NetworkPolicy stops being a convention. The chart renders default-deny. The policy rejects pods in a `workloads=true` namespace that does not already have one. A hand-edited namespace cannot skip the chart and stay open.

### Secrets stay out of git

The app chart renders an `ExternalSecret`. The values file names the key in the secret store. It does not contain the secret.

```yaml
# deploy/values.yaml
externalSecrets:
  - name: api
    key: billing/api
```

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: billing-api
  namespace: billing
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: cluster-store
    kind: ClusterSecretStore
  target:
    name: api
  dataFrom:
    - extract:
        key: billing/api
```

The store itself is an addon, aimed at the cloud secret manager, with credentials that live in the cluster. A pull request that pastes a password into `deploy/` should fail review, and admission can reject a `Secret` whose data did not come from that ExternalSecret if you want the rule to be mechanical. I start with the chart and review. I add the deny once someone bypasses the chart.

### What the composition always sets

Fields a claim author should not choose are absent from the XRD and set in the composition. Encryption, tags, and deletion behavior are the three I refuse to leave as a default of the cloud API.

```yaml
# inside the S3 composition, after the environment is loaded
spec:
  deletionPolicy: Orphan
  forProvider:
    region: {{`{{ $env.region }}`}}
    serverSideEncryptionConfiguration:
      - rules:
          - applyServerSideEncryptionByDefault:
              sseAlgorithm: aws:kms
    tags:
      managed-by: crossplane
      environment: {{`{{ $env.environment }}`}}
```

`Orphan` on a bucket means deleting the claim does not delete the data. Queues use `Delete`, because a queue is cheap to recreate and dangerous to leave behind with a consumer still attached. I write that choice in the composition, per kind, and I say so in the XRD description. Tags make the cloud console answer "who manages this" without a lookup. Encryption is on even when the values file never mentions it.

A claim that tries to set `compositionRef` is rejected by the admission rule above. The platform composition is the only implementation of the API.

### What to watch

The door is safe only while someone looks at the failures. These are the ones I put on a dashboard.

- **Sync failures.** An Application in `SyncFailed` or stuck `Progressing` is usually a missing values file, an admission deny, or a wave that applied a claim before the XRD existed. The message on the Application is the first place to read.
- **Claim reconcile errors.** `Synced=False` or `Ready=False` means the composition rendered an object the cloud rejected, the environment config is stale, or a dependency such as the topic is missing. A claim that retries forever is a name collision, a quota, or a field the XRD still allows and the cloud does not.
- **Drift.** `selfHeal` puts the cluster back to git. It does not, by itself, notice an edit made in the cloud console. Crossplane's `Synced` condition does. I alert on that condition, not only on Argo CD being green.
- **Open namespaces.** A `workloads=true` namespace with no NetworkPolicy is a failed chart render or a namespace created outside the generator. Admission blocks the next pod. A periodic check still pages, because a namespace with no pods is invisible to that rule.
- **Images and secrets that bypassed the path.** A digest that does not match a signed image, or a `Secret` committed next to the chart values, shows up as an admission error on the next sync. If it does not, the policy is in `Audit` and I have not finished the rollout.

## Trade-offs

I recommend this pattern with the costs attached. Teams that adopt it and pretend the costs are not there are the ones who call me back.

**The environment file can go stale.** Compositions trust `aws-network`. When the network changes, that values file has to change with it. Make the file part of the network review.

**Self-service starts after the API exists.** A queue the chart already knows is a values change. A new cloud product is an XRD, a composition, and a chart template. That lead time is the ownership boundary doing its job. If every request still needs a new composition, the platform API is too small.

**Defaults drift.** Schema defaults and chart defaults are two places to express "a queue has a DLQ with five receives." They will disagree. The caller service authors actually use is the chart. Test that path.

**Helm and Go templates share a delimiter.** Escape composition templates inside the chart. The symptom appears on the claim.

**One cluster is one failure domain.** The platform taint keeps a noisy build off the Crossplane nodes. It does not keep an upgrade, a CNI change, or a cluster-scoped network policy from taking Argo CD down together with the services. I accept that while the environment is small enough that a second cluster is more risk than it removes.

**The name prefix lives in the composition.** The XRD describes the rule, and the Go template implements it. A sentence in the schema is cheap insurance. When the prefix or the dependency rule changes, both have to move together, or the console shows two styles of resource.

**Discovery is only as strict as the topic and the AppProject.** The generator will faithfully deploy a repository that should not have been labeled. Spend the time on who can apply `argocd-app`, and on what the `apps` project is allowed to create.

**Default-deny is only half of a connection.** The app chart closes each namespace and opens what the values list. A call still needs an egress rule on the caller and an ingress rule on the callee. One-sided values produce a timeout, which is the signal that the other repository's list is stale. And the policies enforce nothing until the CNI, installed with the cluster, is set to enforce them.

## A management cluster is a later step

What I have described is one cluster per environment. Argo CD, Crossplane, and the workloads share it. The `tier: application` label is on that cluster's Secret, so the generators deploy onto the place they run.

The generators are already a fan-out. They select clusters by label, and each Application's destination is that cluster's server. When a company is ready for a hub, the change is a registration. Run Argo CD on a management cluster, leave `tier: application` off its Secret, and set that label only on the workload clusters. The same ApplicationSets target those clusters. The management cluster stays out of the matrix.

Two details follow from the generators as written. The Crossplane addon uses the same cluster selector, so this change installs providers and XRDs on the workload clusters. Keeping Crossplane on the management cluster means a different selector for that addon. And each workload cluster still needs its own `EnvironmentConfig`, because account, VPC, and OIDC issuer differ.

What the extra cluster buys is a failure domain the node taint cannot provide. Upgrading a workload cluster leaves the Argo CD that deploys to it running. What it does not replace is the logical split: Crossplane for the app, Helm as the package, NetworkPolicy from the same chart, and discovery by label. Those are the lines I want in place first. The hub is an additional line, and the selector is already shaped for it.
{% endraw %}
