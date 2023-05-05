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

## Conclusion

Congratulations! You've reached the end of the cheat-sheet.

Undoubtedly you have some work that will exercise the cluster autoscaler. Go
forth and schedule workloads, and autoscale your nodegroups in good health!

[CDCon GitOpsCon and OSSNA]: https://github.com/kingdonb/eks-cluster#cdcon--gitopscon-and-open-source-summit-na
[Cluster Restore Guide]: https://github.com/kingdonb/eks-cluster#addons-and-authentication
[Flux Bootstrap]: https://github.com/kingdonb/eks-cluster#flux-bootstrap
[Flux monitoring]: https://fluxcd.io/flux/guides/monitoring/
[ebs-csi]: https://github.com/kingdonb/eks-cluster#ebs-csi
