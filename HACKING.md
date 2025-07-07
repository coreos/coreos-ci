# Setting up CoreOS CI

First, create the Jenkins base infra:

```
oc process -l app=coreos-ci \
    -p ENABLE_OAUTH=false \
    -p STORAGE_CLASS_NAME=ocs-storagecluster-ceph-rbd \
    -f https://raw.githubusercontent.com/coreos/fedora-coreos-pipeline/main/manifests/jenkins.yaml | oc create -f -
```

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

### Create ResultsDB authentication secret

Create the ResultsDB authentication secret (available in BitWarden).

```
RDB_USERNAME=username
RDB_PASSWORD=password
oc create secret generic resultsdb-auth \
    --from-literal=username=${RDB_USERNAME} \
    --from-literal=password=${RDB_PASSWORD}
oc label secret/resultsdb-auth \
    jenkins.io/credentials-type=usernamePassword
oc annotate secret/resultsdb-auth  \
    jenkins.io/credentials-description="ResultsDB authentication"
```

### Create Fedora Matrix notifications secret

Create the maubot authentication secret (available in BitWarden under `CoreOS CI Fedora Matrix Auth`).

```
MATRIX_BOT_TOKEN=auth_token
MATRIX_BOT_WEBHOOK_URL=url
oc create secret generic matrix-bot-webhook-token \
    --from-literal=username=${MATRIX_BOT_WEBHOOK_URL} \
    --from-literal=password=${MATRIX_BOT_TOKEN}
oc label secret/matrix-bot-webhook-token \
    jenkins.io/credentials-type=usernamePassword
oc annotate secret/matrix-bot-webhook-token \
    jenkins.io/credentials-description="Token for fedora matrix bot webhook"
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

### Create Dummy root CA certificate secret

The root CA certificate (ca.crt) is required, but not needed for CoreOS CI
so we create a dummy one here.

```
cat <<'EOF' > ca.crt
dummy
EOF
```
Then create the secret:

```
oc create secret generic additional-root-ca-cert \
    --from-literal=filename=ca.crt --from-file=data=ca.crt
oc label secret/additional-root-ca-cert \
    jenkins.io/credentials-type=secretFile
oc annotate secret/additional-root-ca-cert \
    jenkins.io/credentials-description="Dummy Root CA certificate"
```

### Jenkins

Now we can set up the Jenkins S2I builds. We use the same
settings as the FCOS pipeline to ensure that the environment
is the same (notably, Jenkins and plugin versions):

```
oc process -l app=coreos-ci \
    -f https://raw.githubusercontent.com/coreos/fedora-coreos-pipeline/main/manifests/jenkins-images.yaml | oc create -f -
oc process -l app=coreos-ci \
    -f https://raw.githubusercontent.com/coreos/fedora-coreos-pipeline/main/manifests/jenkins-with-cert.yaml | oc create -f -
oc process -l app=coreos-ci \
    -f https://raw.githubusercontent.com/coreos/fedora-coreos-pipeline/main/manifests/jenkins-s2i.yaml | oc create -f -
```

If working on your own fork/branch, you can point the
`JENKINS_S2I_URL` and `JENKINS_S2I_REF` parameters to to
override the repo in which to look for the Jenkins S2I
configuration.

Then start the builds:

```
oc start-build --follow jenkins-with-cert
oc start-build --follow jenkins-agent-base-with-cert
oc start-build --follow jenkins-s2i
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
