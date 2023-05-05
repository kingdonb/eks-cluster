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

[CDCon GitOpsCon and OSSNA]: https://github.com/kingdonb/eks-cluster#cdcon--gitopscon-and-open-source-summit-na
[Cluster Restore Guide]: https://github.com/kingdonb/eks-cluster#addons-and-authentication
[Flux Bootstrap]: https://github.com/kingdonb/eks-cluster#flux-bootstrap
[Flux monitoring]: https://fluxcd.io/flux/guides/monitoring/
