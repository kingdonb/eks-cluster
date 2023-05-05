# Cluster Config

Create the cluster with `eksctl`

```
eksctl create cluster --config-file=cluster.config
```

## CDCon + GitOpsCon and Open Source Summit NA

If you are attending one of the talks, you may skip to the [Roadmap of Talks][]
to find out about some other talks from the author of this guide and friends.

I've collected some details about the environment that runs these demos,
and I'm inviting team members to share this infrastructure in the interest of
[Sustainable Computing][]!

## Addons and Authentication


If you are on my team at Weaveworks and you just want access, already know
about `gsts`, have your AWS account in order, don't need to create another
cluster, want to poke something in this demo env I made, run this command:

```
aws eks --region ca-central-1 update-kubeconfig --name multiarch-ossna23
```

Else, you'll need to take care of some stuff, details of which are below...

Set your account ID in an environment variable:

```
export AWS_PROFILE=sts
export AWS_ACCOUNT_ID=123456789014
export AWS_ROLE_DX=arn:aws:iam::${AWS_ACCOUNT_ID}:role/AdministratorAccess
```

This isn't my real `ACCOUNT_ID`. I'm in the DX group at Weaveworks, that gets
me administrator access to our AWS sub-account. I also needed [gsts][], for my
organization's Google-based IdP that identifies me here.

Google IdP setup is not covered here, as it was already done by corporate AWS
admins (Weaveworks IT department) and I'm very thankful they took care of that.

```
gsts --aws-role-arn "$AWS_ROLE_DX" --aws-region=ca-central-1 --idp-id=$GOOGLE_IDP_ID --sp-id=$GOOGLE_SP_ID
```

We're going to use these values in some later commands. If we set up all these
variables and have our AWS account details in order, we should be able to copy
and paste from this guide and reach the end with all our examples running!

Apologies if something doesn't work; I've tried to provide reference any links,
but YMMV as most of this stuff was [Not invented here][], or so it would seem.

### ebs-csi

To provide persistent storage for the Prometheus data for our monitoring tools,
we'll need access to persistent disks. There's nothing remarkable about this.

Roughly from [AWS Docs][EBS Docs] and the relevant [IAM Docs][EBS IAM Docs]:

```
eksctl create addon --name aws-ebs-csi-driver --cluster multiarch-ossna23 --service-account-role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole --force

eksctl create iamserviceaccount   --name ebs-csi-controller-sa   --namespace kube-system   --cluster multiarch-ossna23   --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy   --approve   --role-only   --role-name AmazonEKS_EBS_CSI_DriverRole
```

You may need to create an IAM OIDC provider first, for this step and/or the
next step. (Or your cluster may already have been provisioned with one.)

The AWS policy document `AmazonEBSCSIDriverPolicy` is covered by the IAM guide
for EBS linked above. This policy permits our CSI driver to attach storage from
the bundled `gp2` storage class, and to clean up when PV bindings are deleted.

### autoscaler

We provisioned our cluster with two node groups, and they may need to scale up.
We certainly don't want to pay for capacity we won't use. The autoscaler addon
can handle both scaling up and scaling down of unused nodes, as well as killing
off workloads safely on nodes that are scheduled for demolition.

Nodes in Kubernetes are ephemeral. We've created two EKS managed node groups.

```
eksctl create iamserviceaccount   --cluster=multiarch-ossna23   --namespace=kube-system   --name=cluster-autoscaler   --attach-policy-arn=arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AmazonEKSClusterAutoscalerPolicy   --override-existing-serviceaccounts   --approve
```

Covered in more detail in the [EKS Autoscaling][] documentation:

```
aws iam create-policy     --policy-name AmazonEKSClusterAutoscalerPolicy     --policy-document file://cluster-autoscaler-policy.json
```

These policies must have an associated IAM OIDC provider in the region, in
order for our cluster's annotations to drive IAM role policy bindings in our
broader (production) AWS account.

### IAM OIDC

If you ran all of the commands above, you were likely prompted at least once to
learn about OIDC. This could have been the first step, avoiding the errors, but
this order invites greater discovery and understanding, fewer questions. "We
need OIDC, but for what? (CSI storage and node auto-scaling, that's what.)"

With all that out of the way, ensure the OIDC provider is associated:

```
eksctl utils associate-iam-oidc-provider --region=ca-central-1 --cluster=multiarch-ossna23 --approve
```

### Flux Bootstrap

If we do not have an exported `GITHUB_TOKEN` with access to the bootstrap repo,
we'll need to create one now. I'm using Flux Community's [github-app-secret][]
to generate time-bound credentials from a GitHub App, that are also scoped to
the single repo. Everyone should use these time-bound, scoped credentials instead
of long-lived (legacy) Personal Access Tokens that may access many repositories.

At the end of the `eksctl create cluster` command, which takes some minutes to
run to completion, we may get an error that our `GITHUB_TOKEN` is not permitted.

#### GitHub App Secret

We need to borrow a fresh token from our GitHub App cronjob; it runs every 30
minutes on the Weave GitOps/CAPI management cluster, where we are creating the
dev clusters most often.

```
kubectx howard-space
kubens default
k get secret my-app-secret -oyaml|yq .data.password|base64 -D|pbcopy && echo "copied GITHUB_TOKEN into clipboard"
export GITHUB_TOKEN=$(pbpaste)
```

#### EKSctl Flux integration

With this GitHub token in our environment, we can now retry the Flux bootstrap:

```
eksctl enable flux --config-file=cluster.config
```

#### SOPS encrypted secrets

There's one more step: our cluster utilizes secrets that have been encrypted in the
repository. We still have the private key, let's put it where Flux can utilize it:

```
export KEY_FP=BF333F5B18A7B7E64ABF8ECD3DA73AD3A17399DC
kubectx multiarch-ossna23
gpg --export-secret-keys --armor "${KEY_FP}" | kubectl create secret generic sops-gpg --namespace=flux-system --from-file=sops.asc=/dev/stdin
flux reconcile ks my-secrets
```

## Roadmap of CDCon/OSS Summit NA Demos

The above steps are providing common shared infrastructure for three presentations
at CDCon/GitOpsCon and Open Source Summit North America 2023:

### GitOpsCon - Lightning Talk

* [Exotic Runtime Targets: Ruby and Wasm on Kubernetes and GitOps Delivery
  Pipelines][GitOpsCon Lightning Talk] - Kingdon Barrett, Weaveworks

So you're a Rubyist, or you're generally interested in how Web Assembly can be
used productively by Rubyists, or by anyone, on Kubernetes?

He's gonna show all that in 10 minutes, with Flux and GitOps? (You bet we are!)

Does Wasm run in Ruby, or is Ruby running in Wasm? Can we get some guidance?

Come to this lightning talk for a brief overview of what a Rubyist GitOps
practitioner learned from surveying the Wasm landscape, and a teaser for our
longer session talks where we'll do some things a bit more useful with Wasm.

### ContainerCon - Session Talk

* [Exotic Runtime Targets: Ruby and Wasm on Kubernetes and GitOps Delivery
  Pipelines][ContainerCon Session Talk] - Kingdon Barrett, Weaveworks 

In our lightning talk yesterday at GitOpsCon, we saw Ruby and Wasm delivered by
GitOps on Kubernetes, with a brief, focused survey of the Wasm landscape.

At ContainerCon, we revisit those lessons to cover 40 minutes, and talk about
what the experience of learning to use Wasm in Ruby on Kubernetes was like.

I developed an actual Kubernetes operator in Ruby, that uses a Wasm module to
handle a part of the job we hypothesized "could be made faster," or something.

Any excuse to explore the new technology, find out what the limitations are,
and consider how by applying these new constraints smartly and tactically we
can ultimately deliver more strategic, long-term benefits, (maybe some we'll
recognize as benefits only after we've already felt their effect for a while!)

What are the pitfalls, and is there any way to re-cast those as new strengths?
Are there "things we need to know" before we've coded ourselves into a corner?

### OpenGovCon - Session Talk

* [Microservices and WASM, Are We There Yet?][OpenGovCon Session Talk] - Will
  Christensen, Defense Unicorns & Kingdon Barrett, Weaveworks

We've all seen hype around WebAssembly. Type safety, reduced binary size, many
language polyglot-supportive, platform-agnostic, but can we actually use it?

We came up with several ideas for how to make productive use of Wasm and used
this as an opportunity to try out some ideas we had been sitting on, and also
attempt to validate our assumptions about this unfamiliar technology.

Why all the hype? Does the sauce actually taste as good as it's made to sound?

Can we actually use Web Assembly to build a Kubernetes operator, or any useful
microservices? And where are we most likely to get stuck, if we cannot today?

[Roadmap of Talks]: https://github.com/kingdonb/eks-cluster#roadmap-of-cdconoss-summit-na-demos
[Sustainable Computing]: https://sustainable-computing.io/
[gsts]: https://github.com/ruimarinho/gsts
[Not invented here]: https://en.wikipedia.org/wiki/Not_invented_here
[EBS Docs]: https://docs.aws.amazon.com/eks/latest/userguide/managing-ebs-csi.html
[EBS IAM Docs]: https://docs.aws.amazon.com/eks/latest/userguide/csi-iam-role.html
[EKS Autoscaling]: https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html
[github-app-secret]: https://github.com/fluxcd-community/github-app-secret

[GitOpsCon Lightning Talk]: https://sched.co/1JpBS
[ContainerCon Session Talk]: https://sched.co/1K55z
[OpenGovCon Session Talk]: https://sched.co/1K57U
