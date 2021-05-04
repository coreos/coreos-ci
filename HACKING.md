# Setting up CoreOS CI

First, create the Jenkins base infra:

```
oc new-app --file=manifests/jenkins.yaml
```

Use the `NAMESPACE` parameter if you're not targeting one
named `coreos-ci`.

If on the production instance, also create the custom local
class PVC:

```
oc create -f manifests/pvc.yaml
```

Otherwise, you can create a vanilla PVC of the same name.

Create the JCasC configmap:

```
oc create configmap jenkins-casc-cfg --from-file=jenkins/config
```

Create the GitHub OAuth secret (these creds live in the
"CoreOS CI" GitHub OAuth App settings) using `oc create -f`:

```
apiVersion: v1
kind: Secret
metadata:
  name: github-oauth
stringData:
  client-id: $ID
  client-secret: $SECRET
```

Create the GitHub webhook shared secret (XXX: jlebon to put
it in the shared secrets repo) using `oc create -f`:

```
apiVersion: v1
kind: Secret
metadata:
  name: github-webhook-shared-secret
stringData:
  secret: $SECRET
```

Create the CoreOS Bot (coreosbot) GitHub token (these creds
correspond to the "CoreOS CI" token of coreobot, with just
`public_repo` and `admin:repo_hook`; XXX: jlebon or bgilbert
to put it in the shared secrets repo):

```
apiVersion: v1
kind: Secret
metadata:
  name: github-coreosbot-token
stringData:
  token: TOKEN
```

Now we can set up the Jenkins S2I builds. We use the same
settings as the FCOS pipeline to ensure that the environment
is the same (notably, Jenkins and plugin versions):

```
oc process -l app=coreos-ci \
    --param "JENKINS_JOBS_URL=https://github.com/coreos/coreos-ci" \
    -f https://raw.githubusercontent.com/coreos/fedora-coreos-pipeline/main/manifests/jenkins-s2i.yaml | oc create -f -
```

If working on your own fork/branch, you can point the
`JENKINS_JOBS_URL` and `JENKINS_JOBS_REF` parameters to
override the repo in which to look for jobs, and/or
`JENKINS_S2I_URL` and `JENKINS_S2I_REF` to override the repo
in which to look for the Jenkins S2I configuration.

Then start a build:

```
oc start-build jenkins
```

And that's it! It's already set up so that jobs will be
created on first boot, etc...
