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
                github {
                    // id must be constant so that job updates work correctly
                    id("fac233be-96f9-4331-851b-6153fdc9af9b")
                    repoOwner('${owner}')
                    repository('${name}')
                    checkoutCredentialsId("github-coreosbot-token-username-password")
                    scanCredentialsId("github-coreosbot-token-username-password")
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
            // things which don't seem to have a nice DSL :(
            configure {
                it / sources / data / 'jenkins.branch.BranchSource' / strategy {
                    properties(class: 'java.util.Arrays\$ArrayList') {
                        a(class: 'jenkins.branch.BranchProperty-array') {
                            'org.jenkinsci.plugins.workflow.multibranch.DurabilityHintBranchProperty' {
                                // we don't care about durability for these CI
                                // jobs. we should be able to interrupt and
                                // restart from scratch whenever
                                hint('PERFORMANCE_OPTIMIZED')
                            }
                        }
                    }
                }
                it / sources / data / 'jenkins.branch.BranchSource' / source << {
                    traits {
                        'org.jenkinsci.plugins.github__branch__source.BranchDiscoveryTrait' {
                            strategyId(1)
                        }
                        'org.jenkinsci.plugins.github__branch__source.OriginPullRequestDiscoveryTrait' {
                            strategyId(1)
                        }
                        'org.jenkinsci.plugins.github__branch__source.ForkPullRequestDiscoveryTrait' {
                            strategyId(1)
                            // allow testing of PRs from project contributors
                            trust(class: 'org.jenkinsci.plugins.github_branch_source.ForkPullRequestDiscoveryTrait\$TrustContributors')
                        }
                    }
                }
                it / sources / data / 'jenkins.branch.BranchSource' / buildStrategies {
                    'jenkins.branch.buildstrategies.basic.ChangeRequestBuildStrategyImpl' {
                        ignoreTargetOnlyChanges(true)
                    }
                    // Skip initial builds during the first branch indexing
                    'jenkins.branch.buildstrategies.basic.SkipInitialBuildOnFirstBranchIndexing' {
                    }
                }
            }
        }
    """
}}
