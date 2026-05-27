/* this job defines all the multibranch jobs for upstream CI repos */

repos = [
    "coreos/afterburn",
    "coreos/bootupd",
    "coreos/console-login-helper-messages",
    "coreos/coreos-assembler",
    "coreos/coreos-installer",
    "coreos/coreos-ci",
    "coreos/coreos-ci-lib",
    "coreos/fedora-coreos-releng-automation",
    "coreos/fedora-coreos-cincinnati",
    "coreos/fedora-coreos-config",
    "coreos/fedora-coreos-pipeline",
    "coreos/ignition",
    "coreos/rpm-ostree",
    "coreos/ssh-key-dir",
    "coreos/zincati",
    "openshift/os",
    "ostreedev/ostree",
    "rhkdump/kdump-utils"
]

node { repos.each { repo ->
    def (owner, name) = repo.split('/')
    jobDsl scriptText: """
        multibranchPipelineJob('${name}') {
            branchSources {
                branchSource {
                    source {
                        github {
                            // id must be constant so that job updates work correctly
                            id("fac233be-96f9-4331-851b-6153fdc9af9b")
                            repoOwner('${owner}')
                            repository('${name}')
                            repositoryUrl('') // remains empty, required by constructor
                            configuredByUrl(false) // required by constructor
                            credentialsId("github-coreosbot-token-username-password")  // replaced both check and scan
                            traits {
                                gitHubBranchDiscovery {
                                    strategyId(1)
                                }
                                gitHubPullRequestDiscovery {
                                    strategyId(1)
                                }
                                gitHubForkDiscovery {
                                    strategyId(1)
                                    trust {
                                        // allow testing of PRs from project contributors
                                        gitHubTrustContributors()
                                    }
                                }
                            }
                        }
                    }
                    strategy {
                        allBranchesSame {
                            props {
                                // we don't care about durability for these CI
                                // jobs. we should be able to interrupt and
                                // restart from scratch whenever
                                durabilityHint {
                                    hint('PERFORMANCE_OPTIMIZED')
                                }
                            }
                        }
                    }
                    buildStrategies {
                        buildChangeRequests {
                            ignoreTargetOnlyChanges(true)
                            ignoreUntrustedChanges(false)
                        }
                    }
                }
            }
            factory {
                workflowBranchProjectFactory {
                    scriptPath('.cci.jenkinsfile')
                }
            }
            orphanedItemStrategy {
                discardOldItems()
            }
            triggers {
                periodicFolderTrigger {
                    // manually rescan once a day; this is important so that it
                    // picks up on deleted branches/PRs which can be cleaned up
                    interval((60 * 24).toString())
                }
            }
        }
    """
}}
