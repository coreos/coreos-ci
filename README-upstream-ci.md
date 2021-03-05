## Upstream CI

### Interacting

#### Logging in

The Jenkins instance uses GitHub authentication. As long as
you are a member of either `coreos`, `ostreedev` or
`openshift`, you should be able to log in and
start/stop/retry jobs.

#### Retrying jobs

In order to retry the failed job, from the GitHub PR, click
through to the details of the failed job and use the retry
"loopy icon" at the top right of the job page.

### Enrolling a new repo for upstream CI

To enroll a repo, simply open a patch to have it added to
the list in `seed-github-ci.Jenkinsfile`.

The `coreosbot` user needs to have admin access to the repo.
This is needed in order to automatically manage webhooks.
Alternatively, you may elect to only provide write access,
at the expense of manually setting up the webhook.

For repos under the `coreos` org. This should be done by
simply adding the `team-bots` team.

### Writing a Jenkinsfile

Upstream repos must have a Jenkins pipeline in a file called
`.cci.jenkinsfile` at the root of the repo.

This pipeline should heavily leverage the custom steps in
[coreos-ci-lib](https://github.com/coreos/coreos-ci-lib).
This is a global shared library which all jobs automatically
have access to.

For example, here is a sample Jenkinsfile to build the
software locally, build FCOS with its results overlayed and
run kola:

```groovy
// Documentation: https://github.com/coreos/coreos-ci/blob/master/README-upstream-ci.md

cosaPod {
    checkout scm
    fcosBuild(make: true)
}
```

The relevant functions used here are
[cosaPod](https://github.com/coreos/coreos-ci-lib/blob/coreos-ci/vars/cosaPod.groovy)
and
[fcosBuild](https://github.com/coreos/coreos-ci-lib/blob/coreos-ci/vars/fcosBuild.groovy).

In practice, it's likely that the `make: true` functionality
in `coreos-ci-lib` will be too simplistic. The `fcosBuild`
step can also take an arbitrary `overlays` parameter to pass
any rootfs directory to overlay.
