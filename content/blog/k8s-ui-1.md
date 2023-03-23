---
title: "Kubernetes Up & Integrated — Authentication (Qubit Eng Blog)"

draft: false
date: "2017-10-27"

description: ""
categories:
    - "Kubernetes"
---

*Note:* This was originally posted on the [Qubit Engineering Blog](https://medium.com/qubit-engineering/kubernetes-up-integrated-authentication-5d2c908c2810), and CopyRight is, at the time of writing, owned by
[Coveo](https://www.coveo.com/en). It was written, and illustrated, by me, but
would have been unreadable but for input and editing by my fantastic teammates.

Note: Much of the information discussed here is considerably out of date, whilst
the broad approaches discussed could still be applied, most of the specific config
and APIs have changed. It is here purely for my personal archive.

This is the first in a four part series on how Qubit have built our
production ready Kubernetes (k8s) environments. There are already many
excellent tutorials on k8s. The title of these articles is an homage to
[Kubernetes Up and Running](http://shop.oreilly.com/product/0636920043874.do),
the (presumably excellent) k8s guide by Brendan Burns, Kelsey Hightower and
Joe Beda. In these articles, we hope to cover the more advanced techniques we
required to make k8s work at “[modern enterprise](https://www.youtube.com/watch?v=iFR_7AKkJFU)” scale.

We hope to cover:

* Authentication and RBAC across multiple cloud providers.
* Secret and configuration management, replacing Puppet with HashiCorp’s
* Vault.
* Ingress, DNS, and migration strategies.
* Observability.

On the way we will briefly mention how the features map to our preexisting
tools and policies, and how k8s has helped us build on them. Code samples
will be provided where possible, and we do hope to open source more of our
tooling in future. I will not hide the warts in our deploy, or the gaps in
the tooling. The goal is to present a realistic picture of the reality of
a Kubernetes deployment at scale.

## Becoming Cloud Agnostic

Qubit have been early adopters of Cloud, PaaS, and container deployments.
We have built features on everything from on-prem Hadoop, AWS Lambda and
RDS, to Google Cloud Platform BigQuery and Dataflow. The on-prem and VM
deploys were traditionally managed by Puppet, and our micro-services were
either deployed directly to VMs, or via containers to Mesos/Marathon
clusters hosted in AWS. As GKE appeared some teams began to take advantage
of it leading to a split, with some internal tooling focused on Marathon,
and various ad hoc deploys on GKE.

Since our micro-services require access to PaaS services from a variety of
cloud providers, we wanted a deployment mechanism that was the same
regardless of the actual underlying cloud provider. This allows us to
co-locate services with their dependencies, simplifying access to PaaS
services and minimising cross cloud data transfer and latency. A common
deployment method also allows us to be more responsive to changes in
contracts/costs as cloud provider offerings change.

As k8s development progressed it became a more obvious choice for a single
platform to target. GKE had largely solved the problem of how to deploy
k8s to Google Cloud (not without its own issues as we shall discuss
later). After a review of the options for AWS k8s deploys we eventually
settled on [kops](https://github.com/kubernetes/kops). Kops allows multi-master, multi-AZ cluster setup, and
management of multiple instance groups. Kops VM base images are well
maintained allowing us to avoid Puppet management of the underlying VMs.
Kops updates, and rolling updates (especially with
`KOPS_FEATURE_FAGS="+DraingAndValidateRollingUpdate"`), actually seem to
behave better than the GKE equivalent (where the master can frequently
become unresponsive during updates).

## Kubernetes Authentication Webhooks

Kubernetes Role Based Access Control (RBAC) was in its early stages during
the beginning of our exploration of production k8s. Our initial attempts
at GKE avoided the subject completely, but it was obvious from Day 1 that
building larger, more efficient, multi-tenant clusters was going to
require RBAC. If you are going to use RBAC you need to be know who is
using the cluster. This is where Authentication comes in.

For GKE we disable all legacy authentication, enable RBAC, and enable IAM
authentication. This is easy, but does have its limits. In particular, the
permissions that can be granted are at a much coarser level than we would
like. We’ll discuss this further when we come to our RBAC configuration.

Enabling RBAC on a kops cluster is easy enough, simply edit the cluster,
update authorization to `rbac: {}`, and roll out the changes. At this time
integrating authentication is not quite as smooth.

K8s API Server has support for authenticating users via a handful of
options:

* File of static usernames/passwords for use via Basic Authentication to
  the API. These are convenient for bootstrapping, and could work for
  small teams, but would require configuration management, and would not
  integrate with an external auth provider.
* File of static tokens. As above.
* Certificates signed by the k8s CA. It is possible to integrate this with
  HashiCorp’s Vault, but that complicates the kops bootstrapping process.
  Vault did not support GCP authentication at the time, the option is now
  there so this may be a more practical solution, though support for
  verifying users Group Membership may not be easy/possible.
* A webhook for verifying bearer tokens presented to the API.

For our AWS clusters, the webhook was the obvious choice. The manner of
configuration is somewhat awkward. A kubectlstyle configuration file must
be placed on each master, the content is as follows.

```yaml
   clusters:
    - name: authn-api
      cluster:
        certificate-authority: /etc/ssl/certs/ca-certificates.crt
        server: https://authn-api/oauth2/k8sTokenReview
   users:
    - name: authn-api
   current-context: webhook
   contexts:
   - context:
      cluster: auth-api
      user: authn-api
    name: webhook
```

   The kops definition must also be updated:

```yaml
   spec:
    kubeAPIServer:
      authenticationTokenWebhookCacheTtl: 2m0s
      authenticationTokenWebhookConfigFile:
        /srv/kubernetes/authn-webhook.yaml
```

Unfortunately, there is no current way to set the content of the yaml
webhook directly in the kops definition. This means that it must be copied
to the server at a known path on each master. Stateful drives are mounted
at different locations on each machine, and other storage is recreated
when masters are updated. This means that when the masters are updated the
file must be copied back up to the known location. Solutions for this are
in the works, and there may be other options that your author has not
considered. Master updates are not so frequent that this has become a
major problem.

## Authentication: Tokens Everywhere

We rely on Google’s G Suite for e-mail and much of our documentation, so
basing our primary authentication on our Google G Suite credentials has
always been the obvious choice. Service Accounts provide a means for us
authenticate between micro-services. Google’s own APIs for JWT signing and
authentication do not include Google Group membership, and only provide
Authorization features for Google’s own APIs and products. For this reason
Qubit have an internal service for signing JWTs, and performing OAuth 2.0
flows. Ours is a relatively simple service, [CoreOS Dex](https://github.com/coreos/dex) would be the
obvious choice for anyone starting out today.

Integrating our existing JWT token authentication with k8s has been a fun
and refreshingly easy experience. The webhook request and response format
can be found in the k8s [documentation](https://kubernetes.io/docs/admin/authentication/%23webhook-token-authentication), and our implementation is simply
a matter of verifying the passed in JWT token, and breaking out the groups
from the JWT claim fields. The process is entirely stateless and could
actually be run as a small service within the cluster. The process is not
particularly latency sensitive, so we actually have the token verification
endpoint on our token issuing/signing service.

The most pleasant aspect of the integration comes from an, almost hidden,
feature of kubectl. If you already use GKE with IAM authentication, and
happen to look in your .kube/config, you will see something like the
following in the users: section.

```yaml
   users:
   - name: gke_project_us-central1-mycluster
    user:
      auth-provider:
        config:
          access-token: ya4......
          cmd-args: config config-helper --format=json
          cmd-path: /home/joe/google-cloud-sdk/bin/gcloud
          expiry: 2017-09-13 15:01:44
          expiry-key: '{.credential.token_expiry}'
          token-key: '{.credential.access_token}'
        name: gcp
```

The access-token and expiry are cached values. The interesting part is the
cmd-args and cmd-path. If name is set to “gcp”, kubectl will take
credentials from the output of executing the given command. The output is
treated as JSON, and the -key fields are JsonPath query expressions into
the resulting data.

If we are devious enough to pretend, for a moment, that “gcp” does not
have to stand for Google Cloud Platform, but could be General Cloudy
Provider, we can leverage this for our own ends. At Qubit we already have
a CLI tool used by our developers for various deployment tasks. It was a
simple task to add an extra command to this to allow the issuing of a JWT
token. When run, the standard OAuth 2.0 browser flow occurs, and all being
well, a signed JWT is retrieved from our API server and dumped out in a
sufficient compatible JSON format.

The end result is that a simple kubectl get podscommand will:

* Run our CLI tool to get a token.
* Verify the user via OAuth.
* Cache the token in kubectl.
* Pass this token, which includes the users groups, as the API token.
* The Authn API server calls the webhook to verify the token.
* The Authn API server verifies the token and informs k8s of the user’s
  group membership.

## Role Based Access Control and Helm

Kubernetes has a powerful and elegant system for configuration of role and
privilege. Users, and groups of users, can be granted various permissions,
either cluster wide via ClusterRoleBindings, or within specific namespaces
via RoleBinding. We will not attempt to describe the details of RBAC here,
but one particular aspect is worth additional attention.

Our goal in configuring RBAC is to give the developer the freedom to work
directly with the k8s API via any tools they wish, whilst simultaneously
trying to provide a degree of protection for services required by our
infrastructure team. We would like teams to be able to share clusters, but
would also like to be able to segregate their jobs when required. The
strategy we have settled on is to give developers full access to
namespaces, based on their Google Cloud group membership. All teams share
access to the default namespace. Teams can have their own namespace, or
namespaces for specific applications. The infrastructure team have a
namespace for running more privileged applications (for instance,
requiring unavoidable, high value secrets). No teams deploy anything to,
or interact with, the kube-system namespace.

There are several choices for how to go about provisioning your services
into k8s clusters. Most tutorials focus on using kubectldirectly, there
are also several templating solutions. For better or worse, we have
settled on [Helm](https://github.com/kubernetes/helm). Helm’s concept of a release, with history and
rollbacks, provides us a nice model for managing all aspects of a deploy.
Rebuilding our existing deployment tool (that targets Docker and
Marathon), to support Helm was relatively straightforward.

One downside of Helm is Tiller. Tiller is a small service deployed in your
cluster which acts as the state store for Helm, keeping the required and
previous configuration state in k8s configmaps. Tiller must be installed
in the cluster, and must have permission to create resources. It is no
longer our users doing the deploying, Tiller does this for them.

By default Tiller is installed to the kube-system namespace. A regular
install would likely give Tiller, and by extension the developer using
Helm to deploy to it, full access to all k8s namespaces. In multi-tenant
clusters that is not acceptable. Fortunately this can be avoided.

Let us say we have a team called team-a. We will create a namespace for
them, and grant them full privileges to a namespace, also called team-a.

```shell
   $ kubectl create namespace team-a
   $ kubectl create rolebinding team-a-admin --namespace team-a \
    --clusterrole=cluster-admin --group=team-a
```

This creates the namespace and assigns the cluster-admin ClusterRole, as a
RoleBinding in the team-a namespace, to members of team-a. It is important
to note that this is a RoleBinding tied to the namespace, not a
ClusterRoleBinding, which would give cluster-admin rights to the entire
cluster. It seems odd but you can bind a ClusterRole with a RoleBinding,
with the subsequent access being allowed only to a specific namespace.

Now, if we deploy Tiller to this namespace, it will get the default
service account. This account does not have sufficient privileges to
manage resources in the namespace. Instead we create a service account for
Tiller, and create it a RoleBinding that binds it to the cluster-admin
ClusterRole in our team-a namespace.

```shell
   $ kubectl create --namespace team-a serviceaccount tiller
   $ kubectl create --namespace team-a rolebinding team-a-tiller \
    --clusterrole=cluster-admin --serviceaccount=team-a:tiller
```

We can now deploy Tiller:

```shell
   $ helm init --upgrade --tiller-namespace team-a \
    --service-account tiller
```

And our developers can finally use it:

```shell
   $ helm --tiller-namespace team-a install alpha/hackernewsthing
```

By restricting our developers to particular namespaces we gain an
additional layer of security, and can also hide some of the guts of our
internal cluster management tooling from them. It gives us more options
for controlling resource quotas on a per-namespace basis, and potentially
segregating work on the underlying VMs via namespaces too (this would
require admission control, but should be possible).

## When is AWS is more GCP than GKE?

One curious aspect of the configuration described above is worth further
discussion. Our own token issuing service fully exposes the group
membership of the user by embedding them in the JWT token. Google’s
equivalent tokens, the ones we use when talking to GKE via IAM, do not.
There are sound technical reasons for this. At Google’s scale, resolving
group memberships is an expensive operation. At our scale it is not. The
practical upshot of this though is that in GKE we cannot use RBAC rules to
limit the namespaces a given user can manage.

In GKE we must grant the developer broad rights to, at the very least, see
the cluster. It is technically possible, via [custom IAM Policies](https://stackoverflow.com/questions/45945074/iam-and-rbac-conflicts-on-google-cloud-container-engine-gke), to
restrict this to a very minimal set of rights, however, our RBAC policies
can only ever match specific users, or everyone that is authenticated, not
groups. The practical upshot is that we must give the developer full
access to the cluster, allowing them full visibility, and management, of
all namespaces. As well as the obvious security implications, this also
exposes the developer to having to see a large number of API objects which
they have no interest in.

We can create smaller clusters, assigning IAM rights to entire clusters,
but to smaller teams, however the extra clusters create an operational
burden, and ultimately are less cost effective in terms of resource usage.

For the moment we are accepting the coarser IAM permissions. Google are
working on this issue. It will be interesting to see how well integrated
into RBAC their final approach will be. In the longer term, if a suitable
solution is not found, we still have the option of building kops clusters
on GCP. The gap in features between a GKE cluster and a kops GCE cluster
is less than one might imagine. As you will see in Ingress and
Observability sections later on, we leverage less GKE features than one
might reasonably expect, and the additional access to direct Kubernetes
API Server features may outweigh the benefits of GKE in the long run.

## Summary

With the configuration described above we have a solid basis for
multi-tenant clusters. Authentication is built on top of a preexisting
company standard, and is relatively consistent between our two cloud
providers. Google G Suite remains our single source of truth for group
membership, and thus authentication and authorisation. kubectl’s cmd-path
feature has allowed us to ensure that the developer’s experience of using
the clusters is in line with what they will find in any Kubernetes
tutorial on-line.
