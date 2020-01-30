This repository will contain OpenShift manifests, Jenkins
configuration as code, job DSLs, and other configs for
powering upstream CI of various CoreOS-related GitHub repos.
The main Jenkins instance is at:

https://jenkins-coreos-ci.apps.ci.centos.org/

It may end up being merged into
https://github.com/coreos/fedora-coreos-pipeline in the
future.

### projectatomic-ci

Upstream CI jobs are currently created manually and running
in https://jenkins-projectatomic-ci.apps.ci.centos.org/ .
Only jlebon and cgwalters can create jobs there (the new
Jenkins will use GitHub OAuth to delegate administration to
the whole CoreOS team).

To create a new job by copying:
- Sign in as jlebon or walters
- `New Item`
- Name it the same as the repository, e.g. `ignition`
- Copy from `rpm-ostree`
- `OK`
- In the job configuration, clear the description and change
  the target repo, e.g. `ignition`
- `Save`

To create a new job without copying:
- Branch Sources -> GitHub
    - Credentials -> coreosbot
    - fill in Owner and Repo
    - Discover pull requests from forks -> Trust ->
      Contributors
    - Add Property -> Pipeline Branch speed/durability
      override -> Performance-optimized
- Build Configuration -> Script Path -> .cci.jenkinsfile
- Scan Multibranch Pipeline Triggers -> Check Periodically
  -> 1 day
- Orphaned Item Strategy -> Discard old items (keep checked)
- `Save`

For push notifications:
- Add a new webhook on the target repo settings:
    - Payload URL: https://jenkins-projectatomic-ci.apps.ci.centos.org/github-webhook/
    - Content Type: application/json
    - Shared Secret: get it from the OpenShift secret
      `github-webhook-shared-secret` in `coreos-ci` or
      `projectatomic-ci`
    - Keep SSL verification enabled
    - Just select `Pull Requests` & `Pushes`
