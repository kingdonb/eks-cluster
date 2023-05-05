# Demo Touchpoints

If you came here by accident, [return to the conference][CDCon GitOpsCon and OSSNA]

## Cheatsheet

### Following the eks-cluster guide notes

#### Teardown

To demolish the cluster, run:

```
cd ~/w/eks-cluster
export AWS_PROFILE=sts
eksctl delete cluster --config-file=cluster.config --force
```

Note that we are destroying several PVs and Load Balancers in the process:

```
$ kg svc|grep LoadBalancer
ingress-nginx                          ingress-public-ingress-nginx-controller                LoadBalancer   10.100.183.19    a260c628d76004a228119f6d3f908a3b-1311778714.ca-central-1.elb.amazonaws.com   80:31229/TCP,443:32314/TCP     24h
vcluster-demo-cluster-00001            vcluster-loadbalancer                                  LoadBalancer   10.100.196.42    a87a19dc0974b488c9f0eead399b0516-1426711807.ca-central-1.elb.amazonaws.com   443:30532/TCP                  18h
vcluster-demo-cluster-00002            vcluster-loadbalancer                                  LoadBalancer   10.100.251.14    a5e6685c9426440fcba3ac12a21db683-1194118517.ca-central-1.elb.amazonaws.com   443:31399/TCP                  18h

$ kg pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                                                                     STORAGECLASS   REASON   AGE
pvc-b077aa34-2a3a-40b8-b143-bb2ade58e944   50Gi       RWO            Delete           Bound    monitoring/prometheus-kube-prometheus-stack-prometheus-db-prometheus-kube-prometheus-stack-prometheus-0   gp2                     24h
pvc-d10522de-b59c-414a-baf8-222036a7a749   5Gi        RWO            Delete           Bound    vcluster-demo-cluster-00001/data-demo-cluster-0                                                           gp2                     17h
pvc-f7c2fafc-807b-4c21-bd67-c34fa66b9e9d   5Gi        RWO            Delete           Bound    vcluster-demo-cluster-00002/data-demo-cluster-2-0                                                         gp2                     17h
```

They are ELBs for Ingress and Control Plane Endpoints, as well as persistent
data for Prometheus datastore and etcd.

DNS is arranged manually on Digital Ocean. The load balancers DNS are here:

* `podinfo-test.hephy.pro` - CNAME maps to `ingress-nginx`
* `xkcd.hephy.pro` - CNAME maps to `ingress-nginx`
* `demo-cluster.hephy.pro` - CNAME maps to `vcluster-demo-cluster-00001`
* `demo-cluster-2.hephy.pro` - CNAME maps to `vcluster-demo-cluster-00002`

Copy the ELB's dns name to the DigitalOcean console when your new Load
Balancers are ready. Tearing down the cluster takes at least 15 minutes.

It will clean up all of these resources as part of the teardown process.

#### Restore EKS cluster and addons

Following the guide at [Cluster Restore Guide][], we'll note any extra steps
for completion that are needed to accomplish the full demo here, if they can't
reasonably be included in the guide upstream due to space and time constraints.

There are several "demo touch-points" that are strategically placed obstacles
that we can leap over, in a way that violates GitOps. We have left a few points
like this in our demo intentionally, so we can show the action of Flagger and
other Kubernetes operators, controllers, or other similar manually executable
workflows. We do this for demonstrative purposes, but it's not suggested that
users will work this way other than in lab scenarios for better understanding.

Starting from the guide:

#### `eksctl create cluster`

```
eksctl create cluster --config-file=cluster.config
```

This is the first instruction in the guide, and since you've just torn down a
cluster this one will already have the AWS credentials that you need in place.

The config file is the same one in the repository root here.

It may take a few moments longer than `eksctl delete` for CloudFormation to
finish cleaning up after itself.

```
2023-05-05 12:01:12 [✖]  creating CloudFormation stack "eksctl-multiarch-ossna23-cluster": operation error CloudFormation: CreateStack, https response error StatusCode: 400, RequestID: 9148a40b-3776-457e-b38a-320dec0a2e52, AlreadyExistsException: Stack [eksctl-multiarch-ossna23-cluster] already exists
$ eksctl delete cluster --region=ca-central-1 --name=multiarch-ossna23
Error: unable to describe cluster control plane: operation error EKS: DescribeCluster, https response error StatusCode: 404, RequestID: 033adcca-0276-4793-8dbc-e3109f4704b7, ResourceNotFoundException: No cluster found for name: multiarch-ossna23.
```

Now it is cleaned up. Create from the config file, and wait another 15 minutes
for this process to complete at least. It will end with a failure signifying
that Flux has not been bootstrapped on the cluster. This is touch-point number
two, where we can stop to talk about some things now that cluster is created.

#### Flux Bootstrap

There are some topics mentioned in the upstream README.md where we expect demo
to go a bit sideways until we have handled the prerequisites. But we can allow
Flux's health checking to inform us about the problems, and take our cues from
Flux's own status reporting that we have thoughtfully constructed through repo
structure to illuminate the issues as they arise and avoid many spurious errors
we might otherwise encounter through sane application of dependency ordering.

After about 15 minutes, if all goes well, EKS cluster is ready from the config.

```
Please enter your GitHub personal access token (PAT): ✗ could not read token: could not read from stdin: EOF
2023-05-05 12:18:57 [ℹ]  Flux v2 failed to install successfully. check configuration and re-run `eksctl enable flux`
Error: running Flux Bootstrap: exit status 1
```

Let's clean up the terminal by renaming the context with `kconf`, and follow
the [Flux Bootstrap][] instructions from the bottom of our EKS Cluster guide:

```
kconf rename kingdon@weave.works@multiarch-ossna23.ca-central-1.eksctl.io multiarch-ossna23
kubectx howard-space
kubens default
k get secret my-app-secret -oyaml|yq .data.password|base64 -D|pbcopy && echo "copied GITHUB_TOKEN into clipboard"
export GITHUB_TOKEN=$(pbpaste)
kubectx multiarch-ossna23
eksctl enable flux --config-file=cluster.config
```

##### Flux and SOPS

And let's ensure that Flux, now bootstrapped successfully, has access to the
SOPS secrets needed for Prometheus's slack reporter.

```
export KEY_FP=BF333F5B18A7B7E64ABF8ECD3DA73AD3A17399DC
kubectx multiarch-ossna23
gpg --export-secret-keys --armor "${KEY_FP}" | kubectl create secret generic sops-gpg --namespace=flux-system --from-file=sops.asc=/dev/stdin
flux reconcile ks my-secrets
```

##### Flux and Prometheus

The next failure is from Prometheus Kube:

```
$ kg ks
...
flux-system   prometheus-kube     65s   False     Health check failed after 1m0.033803832s: timeout waiting for: [Kustomization/monitoring/loki-stack status: 'InProgress', Kustomization/monitoring/monitoring-config status: 'InProgress', Kustomization/monitoring/monitoring-stack status: 'InProgress']
```

There is nothing to do about it, as the dependencies from kube-prometheus-stack
and the [Flux monitoring][] guide must be deployed in order. Just wait a minute.

...

```
$ kg ks
NAMESPACE     NAME                AGE     READY   STATUS
flux-system   cert-issuers        7m38s   True    Applied revision: main@sha1:0e81ec85a4f98c89d7b616b4a2292d14cd1d6a52
flux-system   cert-manager        7m38s   True    Applied revision: main@sha1:0e81ec85a4f98c89d7b616b4a2292d14cd1d6a52
flux-system   flagger             7m38s   True    Applied revision: main@sha1:0e81ec85a4f98c89d7b616b4a2292d14cd1d6a52
flux-system   flux-system         7m51s   True    Applied revision: main@sha1:0e81ec85a4f98c89d7b616b4a2292d14cd1d6a52
flux-system   keda                7m38s   True    Applied revision: main@sha1:0e81ec85a4f98c89d7b616b4a2292d14cd1d6a52
flux-system   my-secrets          7m38s   True    Applied revision: main@sha1:0e81ec85a4f98c89d7b616b4a2292d14cd1d6a52
flux-system   nginx               7m38s   True    Applied revision: main@sha1:0e81ec85a4f98c89d7b616b4a2292d14cd1d6a52
flux-system   podinfo             7m38s   True    Applied revision: main@sha1:0e81ec85a4f98c89d7b616b4a2292d14cd1d6a52
flux-system   prometheus-kube     7m38s   True    Applied revision: main@sha1:0e81ec85a4f98c89d7b616b4a2292d14cd1d6a52
monitoring    loki-stack          7m37s   True    Applied revision: ossna23@sha1:6a76f382613adffa3a3e1a45de6cda6314da0a56
monitoring    monitoring-config   7m37s   True    Applied revision: ossna23@sha1:6a76f382613adffa3a3e1a45de6cda6314da0a56
monitoring    monitoring-stack    7m37s   True    Applied revision: ossna23@sha1:6a76f382613adffa3a3e1a45de6cda6314da0a56

$ kg hr
NAMESPACE       NAME                    AGE     READY   STATUS
ingress-nginx   public-ingress          4m30s   True    Release reconciliation succeeded
keda-test       http-addon              7m45s   True    Release reconciliation succeeded
keda-test       keda                    7m45s   True    Release reconciliation succeeded
monitoring      flagger                 7m45s   True    Release reconciliation succeeded
monitoring      kube-prometheus-stack   7m43s   True    Release reconciliation succeeded
monitoring      loki-stack              5m53s   True    Release reconciliation succeeded
```

That all worked itself out, see?

##### Flux Example Apps - DNS

Now update the DNS for the load balancers that were provisioned:

* `podinfo-test.hephy.pro`
* `xkcd.hephy.pro`

Use the DNS record from the LoadBalancer:

```
$ kg svc|grep LoadBalancer
ingress-nginx   ingress-public-ingress-nginx-controller                LoadBalancer   10.100.129.133   a3faf329a89444425a990f87c430ea46-545149846.ca-central-1.elb.amazonaws.com   80:30597/TCP,443:32636/TCP     5m6s
```

##### Flux and Cert-Manager

Set each of those CNAMEs to point at the ingress controller. We know it has
worked when we see valid certificates from our ingress and cert-manager:

```
$ kg cert -w
NAMESPACE   NAME          READY   SECRET        AGE
test        podinfo-tls   True    podinfo-tls   60s
```

This may take 5-10 minutes, but it should not take longer as the TTL for these
CNAMEs has been set to only `600s`. It could also fail due to rate-limiting if
you do this too often. (More than 5x per week!)

In that case, the best thing to do is pick a new name, or add an additional
name. It's easy to bypass this rate limit if you have experienced this before.

```
  Warning  Failed     6m19s  cert-manager-challenges  Accepting challenge authorization failed: acme: authorization error for podinfo-test.hephy.pro: 400 urn:ietf:params:acme:error:dns: DNS problem: NXDOMAIN looking up A for podinfo-test.hephy.pro - check that a DNS record exists for this domain; no valid AAAA records found for podinfo-test.hephy.pro
```

In case of brief temporary failures, delete the certificate and try again!

#### EKSctl: Persistent Disks for Prometheus

We've come to the [ebs-csi][] configuration, which has a few prerequisites.

```
export AWS_ACCOUNT_ID=4574....
eksctl utils associate-iam-oidc-provider --region=ca-central-1 --cluster=multiarch-ossna23 --approve
eksctl create iamserviceaccount   --name ebs-csi-controller-sa   --namespace kube-system   --cluster multiarch-ossna23   --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy   --approve   --role-only   --role-name AmazonEKS_EBS_CSI_DriverRole
eksctl create addon --name aws-ebs-csi-driver --cluster multiarch-ossna23 --service-account-role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole --force
```

The `AWS_ACCOUNT_ID` is from the Notion doc "Accessing AWS Resources" that you
may have received in your onboarding materials; it is department-specific as we
each have our own separate roles in the company granting administrator access.

(The OIDC provider should already exist in the region, hopefully!)

It's `AWS_ROLE_CX` or something like `AWS_ROLE_SANDBOX`, the account ID is just
the numbers from in between the bookends: `arn:aws:iam::{{right here}}:role`.
It should take only about `30s` to provision the addon. The good news should
come in about 3-5 minutes, that Prometheus has a persistent disk attached and
it can begin collecting data:

```
$ k describe pvc -n monitoring prometheus-kube-prometheus-stack-prometheus-db-prometheus-kube-prometheus-stack-prometheus-0
...
  Normal   ExternalProvisioning  2m47s (x283 over 72m)  persistentvolume-controller  waiting for a volume to be created, either by external provisioner "ebs.csi.aws.com" or manually created by system administrator
  Warning  ProvisioningFailed    70s                   ebs.csi.aws.com_ebs-csi-controller-7949bff697-l2bgb_de8f3ab6-3025-4d5e-8220-6adc43a53870  (combined from similar events): failed to provision volume with StorageClass "gp2": rpc error: code = Internal desc = Could not create volume "pvc-d8e7bc4a-e86a-47ae-ad5b-63cc8918c948": could not create volume in EC2: UnauthorizedOperation: You are not authorized to perform this operation. Encoded authorization failure message: G2B5CCkIzzBQ5qig6MlsMFhpqR8-Kf9yTAZVLglY4TolhFjWDb3p6R-J9vWcWp7dNz8YRThnzOEDa0n7aEIq0f8_sE899dSKFmp4nHs5rEJGChNN6hEGNjJ-_g4UF9SNr5vCgI0PoeG34t8dAHjwU0EB8b6eFzKZhZ5yaNeAaibBrqQ4bhuRjZ_9VDYM-nna3-SiXPI7dr_wSuf9ktoGcK-bz2gzqZFOuEUeNQpT-G44gdyA6kt_TQwZ9f7TqIcv-rmLeLSJU5EYtWfL-kHIIMpvVetwwAULvKBa9KUchAUI8I8p6hYkIu4fttSRS2PwT4i88yjTiJNzb8QRh5OkAwG_INM3z-EKOIMNCApt3-qUbvAvos3dol-T6756IQxd01NIqdSHI5VX_Wl2BapgjG_W2cHUKu6_DWM0_KEmW_Snwn6dnWjRqkPyGFHmxCJpyG_oAjCMD6jBfKMknTWVueldV1lXXBtlN9UKZ20bMcA_HzgvXCJ7SCAwollKQBfwSOWNZnZ84xjMcZJkF5wNC62VperlawYiUxiHKH1FEIoxW6VPGTrSwiQBdxWK4CDLdmGSKNFcw6kO97egnbOKyoZHFpmsupAJdtQwKyeltwIQsLBnBCbOpIolbMU7rTzbbdkbNmjUSu941gqgRWfAq3sfx9kPKdExBw

$ ks rollout restart deploy/ebs-csi-controller deploy/ebs-csi-node
deployment.apps/ebs-csi-controller restarted
$ ks rollout restart ds/ebs-csi-node
daemonset.apps/ebs-csi-node restarted
```

If you get these errors, time to delete the addon and try again:

```
eksctl delete addon --name aws-ebs-csi-driver --cluster multiarch-ossna23
eksctl create addon --name aws-ebs-csi-driver --cluster multiarch-ossna23 --service-account-role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole --force
```

The addon may have been created after the service account. Now it works:

```
...
  Normal   ProvisioningSucceeded  76s  ebs.csi.aws.com_ebs-csi-controller-7949bff697-gcvzj_99911d21-c761-40e4-b271-a214ba2346c1  Successfully provisioned volume pvc-d8e7bc4a-e86a-47ae-ad5b-63cc8918c948

$ kg pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                                                                     STORAGECLASS   REASON   AGE
pvc-d8e7bc4a-e86a-47ae-ad5b-63cc8918c948   50Gi       RWO            Delete           Bound    monitoring/prometheus-kube-prometheus-stack-prometheus-db-prometheus-kube-prometheus-stack-prometheus-0   gp2                     2m32s

$ k -n monitoring get po
NAME                                                       READY   STATUS    RESTARTS      AGE
alertmanager-kube-prometheus-stack-alertmanager-0          2/2     Running   1 (80m ago)   80m
flagger-59b8594797-94vch                                   1/1     Running   0             81m
kube-prometheus-stack-grafana-77966dd8bd-j97f5             3/3     Running   0             80m
kube-prometheus-stack-kube-state-metrics-cb48bd494-xfljk   1/1     Running   0             80m
kube-prometheus-stack-operator-758b68bd5f-vrkd9            1/1     Running   0             80m
kube-prometheus-stack-prometheus-node-exporter-46phl       1/1     Running   0             80m
kube-prometheus-stack-prometheus-node-exporter-l9fhw       1/1     Running   0             80m
loki-stack-0                                               1/1     Running   0             80m
loki-stack-promtail-8ggqv                                  1/1     Running   0             80m
loki-stack-promtail-gmnw5                                  1/1     Running   0             80m
prometheus-kube-prometheus-stack-prometheus-0              2/2     Running   0             80m
```

Now we should have basically everything we need for simple Flagger demos.
But...

We've one more AWS thing to take care of before we can get on with big demos:

#### EKSctl: Autoscaling Managed Node Groups

Our cluster autoscaler is in crashloop:

```
$ ks get po
...
cluster-autoscaler-7b8db694f6-c6dvq   0/1     CrashLoopBackOff   21 (26s ago)   84m
```

We need to do similar role binding magic as all that we've just done for EBS
persistent disks. There are a few minutes until Prometheus detects the error
condition and starts alerting about it in the [#ossna23-demo slack channel](https://empfin.slack.com/archives/C0563R1D1H8).

We also need to suspend Flux because Flux is managing the annotation, and we're
about to create a new role that doesn't match the one in Git. We'll update it.

```
$ flux suspend ks flux-system
# If you already created out of order, delete the iam service account:
## $ eksctl delete iamserviceaccount --cluster=multiarch-ossna23 --namespace=kube-system --name=cluster-autoscaler
# Then create it again

$ eksctl create iamserviceaccount   --cluster=multiarch-ossna23   --namespace=kube-system   --name=cluster-autoscaler   --attach-policy-arn=arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AmazonEKSClusterAutoscalerPolicy   --override-existing-serviceaccounts   --approve
$ ks rollout restart deploy/cluster-autoscaler
deployment.apps/cluster-autoscaler restarted
$ ks get po -w
NAME                                  READY   STATUS    RESTARTS   AGE
...
cluster-autoscaler-7b8db694f6-hsqsl   1/1     Running   0          7s
$ ks edit sa cluster-autoscaler
...

apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::457472006214:role/eksctl-multiarch-ossna23-addon-iamserviceacc-Role1-GK446CZHH6HE
```

In about 35 seconds this `eksctl` command completes, and we can see the cluster
autoscaler pod running uninterrupted on the next successful restart.

Harvest the newly created role annotation and use it to replace ours in Git:

```
$ cd ~/w/fleet-infra/clusters/multiarch-ossna
$ vi infra/autoscaler/cluster-autoscaler-autodiscover.yaml
# (update the annotation value)
$ git commit ...
$ git push
```

When the gitrepo is updated, safe to resume `flux-system` without interrupting
the autoscaler that we have just configured:

```
$ kg gitrepo
NAMESPACE     NAME          URL                                           AGE    READY   STATUS
flux-system   flux-system   ssh://git@github.com/kingdon-ci/fleet-infra   101m   True    stored artifact for revision 'main@sha1:d3ef057dcbbf10037b3104ae7cec2984f3aa49fd'

# (note that the revision is the one we just pushed)

$ flux resume ks flux-system
```

... and I had to delete and recreate the service account one more time after this.
I must have got the order wrong. The failing (and then eventual successful) output
of cluster-autoscaler looks like all this:

```
E0505 18:09:38.943755       1 aws_manager.go:262] Failed to regenerate ASG cache: WebIdentityErr: failed to retrieve credentials
caused by: AccessDenied: Not authorized to perform sts:AssumeRoleWithWebIdentity
	status code: 403, request id: a92dc438-f4b9-42d4-af1b-bea8f99ce16c

...

I0505 18:15:22.234276       1 aws_manager.go:266] Refreshed ASG list, next refresh after 2023-05-05 18:16:22.23427234 +0000 UTC m=+81.277883299
...
I0505 18:15:42.258737       1 static_autoscaler.go:492] Calculating unneeded nodes
I0505 18:15:42.258759       1 pre_filtering_processor.go:66] Skipping ip-192-168-30-102.ca-central-1.compute.internal - node group min size reached
I0505 18:15:42.258775       1 pre_filtering_processor.go:66] Skipping ip-192-168-82-229.ca-central-1.compute.internal - node group min size reached
```

When the created role and Git are in sync, the ASG list can be read and the
nodes in each node group are considered to determine if they are still needed.

(Node groups will not scale below 1 node, but it can be configured if needed.)

## Conclusion

Congratulations! You've reached the end of the cheat-sheet.

Undoubtedly you have some work that will exercise the cluster autoscaler. Go
forth and schedule workloads, and autoscale your nodegroups in good health!

### Bonus: KEDA

We would like to scale pods to zero when they are not needed. KEDA provides an
HTTP add-on and an example that we can use to test `HTTPScaledObject` behavior.

```
$ cd ~/w/fleet-infra
$ cd clusters/multiarch-ossna/keda/http-add-on/
$ helm install xkcd ./examples/xkcd  -n keda-test
```

If you visit [xkcd.hephy.pro][] now, you should find that KEDA's XKCD-serv is
responding... after a few seconds of waiting, in case you hit a cold start!

```
$ k -n keda-test get ingress
NAME   CLASS    HOSTS            ADDRESS                                                                     PORTS   AGE
xkcd   public   xkcd.hephy.pro   a3faf329a89444425a990f87c430ea46-545149846.ca-central-1.elb.amazonaws.com   80      53s
```

This uses the load balancer we provisioned earlier, way back in the section
[Flux Example Apps - DNS][]. If you've made it this far, then pat yourself on
the back, as we've now configured and tested our autoscalers from end-to-end!

If you run into trouble, you may need to build a multi-arch image as your
XKCD-serv could land on either amd64 or aarch64 environment at runtime.
See [the issue][kedacore/http-add-on#665] for more information on this.

There's further discussion of specific innovative solutions to the runtime
environment problem in the [CDCon GitOpsCon and OSSNA][] [Roadmap of Talks][].
We hope you enjoyed the session!

### Bonus: Cluster-API and VCluster

We've got some VCluster definitions that are for CAPI. We need CAPI installed
in order to take advantage of them. We may use these VClusters as a clean slate
from which we can inherit some of the infrastructure that we just provisioned.

```
$ clusterctl init --infrastructure vcluster
Fetching providers
Skipping installing cert-manager as it is already installed
Installing Provider="cluster-api" Version="v1.4.2" TargetNamespace="capi-system"
Installing Provider="bootstrap-kubeadm" Version="v1.4.2" TargetNamespace="capi-kubeadm-bootstrap-system"
Installing Provider="control-plane-kubeadm" Version="v1.4.2" TargetNamespace="capi-kubeadm-control-plane-system"
Installing Provider="infrastructure-vcluster" Version="v0.1.3" TargetNamespace="cluster-api-provider-vcluster-system"

Your management cluster has been initialized successfully!

You can now create your first workload cluster by running the following:

  clusterctl generate cluster [name] --kubernetes-version [version] | kubectl apply -f -

$ cd ~/vcluster-moo-cluster
```

It takes two or three minutes for Cluster-API to spin up its providers, then we
can use declarative definition to create two more virtual demo clusters on AWS.

[This repo][kingdonb/vcluster-moo-cluster] contains the Cluster-API definitions
and some Load Balancers, they are not managed by GitOps as they are cattle and
can be killed off at any time.

```
k create -f vclusters/vcluster-demo-cluster.yaml; k create -f vclusters/vcluster-demo-cluster-2.yaml
namespace/vcluster-demo-cluster-00001 created
cluster.cluster.x-k8s.io/demo-cluster created
vcluster.infrastructure.cluster.x-k8s.io/demo-cluster created
namespace/vcluster-demo-cluster-00002 created
cluster.cluster.x-k8s.io/demo-cluster-2 created
vcluster.infrastructure.cluster.x-k8s.io/demo-cluster-2 created
```

They will also need load balancers, and the load balancers need DNS:

```
$ k create -f load-balancers/vcluster-demo-cluster-00001-load-balancer.yaml; k create -f load-balancers/vcluster-demo-cluster-00002-load-balancer.yaml
service/vcluster-loadbalancer created
service/vcluster-loadbalancer created

$ kg svc|grep LoadBalancer
ingress-nginx                          ingress-public-ingress-nginx-controller                           LoadBalancer   10.100.129.133   a3faf329a89444425a990f87c430ea46-545149846.ca-central-1.elb.amazonaws.com    80:30597/TCP,443:32636/TCP     135m
vcluster-demo-cluster-00001            vcluster-loadbalancer                                             LoadBalancer   10.100.120.93    a960c27386d2341dd9b2e2ecfa8a6105-545386275.ca-central-1.elb.amazonaws.com    443:30587/TCP                  18s
vcluster-demo-cluster-00002            vcluster-loadbalancer                                             LoadBalancer   10.100.175.26    a94d076e89489453ba71bb7c1c0ecb2e-1561788663.ca-central-1.elb.amazonaws.com   443:30461/TCP                  16s
```

Create the DNS records in DigitalOcean by hand:

* `demo-cluster.hephy.pro`
* `demo-cluster-2.hephy.pro`

Give the changes a few seconds to propagate, then try a connection with the
`vcluster` cli:

```
$ for i in demo-cluster demo-cluster-2; do kconf rm $i ;done
$ vcluster connect demo-cluster -n vcluster-demo-cluster-00001 --server=https://demo-cluster.hephy.pro
done √ Switched active kube context to vcluster_demo-cluster_vcluster-demo-cluster-00001_multiarch-ossna23
- Use `vcluster disconnect` to return to your previous kube context
- Use `kubectl get namespaces` to access the vcluster

$ kg po
NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE
kube-system   coredns-6f8f9bbf68-k2mh4   1/1     Running   0          2m44s

$ kubectx multiarch-ossna23
$ vcluster connect demo-cluster-2 -n vcluster-demo-cluster-00002 --server=https://demo-cluster-2.hephy.pro
$ kconf rename vcluster_demo-cluster_vcluster-demo-cluster-00001_multiarch-ossna23 demo-cluster
$ kconf rename vcluster_demo-cluster-2_vcluster-demo-cluster-00002_multiarch-ossna23 demo-cluster-2

$ kubectx demo-cluster-2
✔ Switched to context "demo-cluster-2".
$ kg po
NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE
kube-system   coredns-6f8f9bbf68-992k2   1/1     Running   0          6m33s
```

Now we are really ready to do some damage! Demo clusters that we can provision
and connect to a load balancer of our own design, in just a few easy seconds.

Really hope you have enjoyed this, please visit [Flux at GitOpsCon/OSS 2023][]
or return to the [CDCon GitOpsCon and OSSNA][] page, and [Roadmap of Talks][]!

[CDCon GitOpsCon and OSSNA]: https://github.com/kingdonb/eks-cluster#cdcon--gitopscon-and-open-source-summit-na
[Cluster Restore Guide]: https://github.com/kingdonb/eks-cluster#addons-and-authentication
[Flux Bootstrap]: https://github.com/kingdonb/eks-cluster#flux-bootstrap
[Flux monitoring]: https://fluxcd.io/flux/guides/monitoring/
[ebs-csi]: https://github.com/kingdonb/eks-cluster#ebs-csi
[xkcd.hephy.pro]: https://xkcd.hephy.pro
[Flux Example Apps - DNS]: https://github.com/kingdonb/eks-cluster/blob/main/CHEATSHEET.md#flux-example-apps---dns
[kedacore/http-add-on#665]: https://github.com/kedacore/http-add-on/issues/665
[Roadmap of Talks]: https://github.com/kingdonb/eks-cluster#roadmap-of-cdconoss-summit-na-demos
[kingdonb/vcluster-moo-cluster]: https://github.com/kingdonb/vcluster-moo-cluster
[Flux at GitOpsCon/OSS 2023]: https://bit.ly/gitopscon2023
