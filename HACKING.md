# Setting up CoreOS CI

First, create the Jenkins base infra:

```
oc process -l app=coreos-ci \
    -p ENABLE_OAUTH=false \
    -p NAMESPACE=coreos-ci \
    -p STORAGE_CLASS_NAME=ocs-storagecluster-ceph-rbd \
    -f https://raw.githubusercontent.com/coreos/fedora-coreos-pipeline/main/manifests/jenkins.yaml | oc create -f -
```

Change the `NAMESPACE` parameter if you're not targeting one
named `coreos-ci`.

We turn off the default OpenShift authentication because we
will use GitHub instead.

### Paste the GitHub OAuth secrets

The GitHub OAuth plugin does not support Jenkins credentials
so we directly substitute the secret into the JCasC
configmap. The secret is available in BitWarden.

```
CLIENT_ID=<SECRET>
CLIENT_SECRET=<SECRET>
sed -i -e "s,GITHUB_OAUTH_CLIENT_ID,${CLIENT_ID}," jenkins/config/github-oauth.yaml
sed -i -e "s,GITHUB_OAUTH_CLIENT_SECRET,${CLIENT_SECRET}," jenkins/config/github-oauth.yaml
```

### Create the JCasC configmap

```
oc create configmap jenkins-casc-cfg --from-file=jenkins/config
```

### Create the GitHub webhook shared secret

The current webhook shared secret used is in BitWarden:

```
# use `echo -n` to make sure no newline is in the secret
echo -n "$SECRET" > secret
oc create secret generic github-webhook-shared-secret --from-file=text=secret
oc label secret/github-webhook-shared-secret \
    jenkins.io/credentials-type=secretText
oc annotate secret/github-webhook-shared-secret \
    jenkins.io/credentials-description="GitHub Webhook Shared Secret"
```

### Create coreosbot GitHub token secrets

Create the CoreOS Bot (coreosbot) GitHub token secret (this
corresponds to the "CoreOS CI" token of coreosbot, with just
`public_repo` and `admin:repo_hook`; these creds are
available in BitWarden).

We create two secrets here. One as a usernamePassword (used by the upstream
GitHub jobs) and one secretText (used by the GitHub Plugin):

```
TOKEN=<TOKEN>
oc create secret generic github-coreosbot-token-text --from-literal=text=${TOKEN}
oc label secret/github-coreosbot-token-text \
    jenkins.io/credentials-type=secretText
oc annotate secret/github-coreosbot-token-text \
    jenkins.io/credentials-description="GitHub coreosbot token as text/string"

oc create secret generic github-coreosbot-token-username-password \
    --from-literal=username=coreosbot \
    --from-literal=password=${TOKEN}
oc label secret/github-coreosbot-token-username-password \
    jenkins.io/credentials-type=usernamePassword
oc annotate secret/github-coreosbot-token-username-password  \
    jenkins.io/credentials-description="GitHub coreosbot token as username/password"
```

### Create pipeline configmap

Run:

```
oc process -l app=coreos-ci \
    --param "JENKINS_JOBS_URL=https://github.com/coreos/coreos-ci" \
    -f https://raw.githubusercontent.com/coreos/fedora-coreos-pipeline/main/manifests/pipeline.yaml | oc create -f -
```

If working on your own fork/branch, you can point the
`JENKINS_JOBS_URL` and `JENKINS_JOBS_REF` parameters to
override the repo in which to look for jobs.

### Jenkins

Now we can set up the Jenkins S2I builds. We use the same
settings as the FCOS pipeline to ensure that the environment
is the same (notably, Jenkins and plugin versions):

```
oc process -l app=coreos-ci \
    -f https://raw.githubusercontent.com/coreos/fedora-coreos-pipeline/main/manifests/jenkins-s2i.yaml | oc create -f -
```

If working on your own fork/branch, you can point the
`JENKINS_S2I_URL` and `JENKINS_S2I_REF` parameters to to
override the repo in which to look for the Jenkins S2I
configuration.

Then start a build:

```
oc start-build jenkins-s2i
```

And that's it! It's already set up so that jobs will be
created on first boot, etc...

## Moving CoreOS CI: GitHub OAuth

If moving to a new location, you'll need an owner of the
`coreos` GitHub org to update the callback URL of the CoreOS
CI GitHub app to point to the new Jenkins instance as
described in <https://plugins.jenkins.io/github-oauth/>. For
example:

https://jenkins-coreos-ci.apps.ocp.fedoraproject.org/securityRealm/finishLogin
