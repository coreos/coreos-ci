This repository will contain OpenShift manifests, Jenkins
configuration as code, job DSLs, and other configs for
powering upstream CI of various CoreOS-related GitHub repos.
The main Jenkins instance is at:

https://jenkins-coreos-ci.apps.ci.centos.org/

It may end up being merged into
https://github.com/coreos/fedora-coreos-pipeline in the
future.

CoreOS CI uses the same S2I builder defined in the
[Fedora CoreOS pipeline](https://github.com/coreos/fedora-coreos-pipeline/tree/master/jenkins/master).
All build config and plugin changes should happen there.

### Enrolling a new repo for upstream CI

To enroll a repo, simply open a patch to have it added to
the list in `seed-github-ci.Jenkinsfile`.

### Adding a job

While CoreOS CI is today mostly focused on GitHub PR
testing, it is flexible enough to contain other related
jobs.

Any Jenkinsfile added in the `jobs/` directory will be
converted into a pipeline job.

### Repo structure

The `jenkins/config` directory contains
[Configuration as Code](https://github.com/jenkinsci/configuration-as-code-plugin)
fragments to configure Jenkins on startup.

The `jobs/` directory contains Jenkinsfiles which are
converted to pipeline jobs.

The `manifests/` directory contains the OpenShift template
for creating Jenkins itself.
