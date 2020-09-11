/* this job defines all the multibranch jobs for upstream CI repos */

repos = [
    "coreos/afterburn",
    "coreos/console-login-helper-messages",
    "coreos/coreos-assembler",
    "coreos/coreos-installer",
    "coreos/coreos-ci",
    "coreos/coreos-ci-lib",
    "coreos/fcct",
    "coreos/fedora-coreos-releng-automation",
    "coreos/fedora-coreos-cincinnati",
    "coreos/fedora-coreos-config",
    "coreos/fedora-coreos-pipeline",
    "coreos/fedora-coreos-streams",
    "coreos/ignition",
    "coreos/ignition-dracut",
    "coreos/rpm-ostree",
    "coreos/ssh-key-dir",
    "coreos/zincati",
    "ostreedev/ostree"
]

node { repos.each { repo ->
    def (owner, name) = repo.split('/')
    jobDsl scriptText: """
        folder('github-ci')
        folder('github-ci/${owner}')
        multibranchPipelineJob('github-ci/${repo}') {
            branchSources {
                github {
                    // id must be constant so that job updates work correctly
                    id("fac233be-96f9-4331-851b-6153fdc9af9b")
                    repoOwner('${owner}')
                    repository('${name}')
                    checkoutCredentialsId("github-coreosbot-token")
                    scanCredentialsId("github-coreosbot-token")
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
                // manually rescan once a day; this is important so that it
                // picks up on deleted branches/PRs which can be cleaned up
                periodic(60 * 24)
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
                        'jenkins.plugins.git.traits.WipeWorkspaceTrait' {
                          extension(class: 'hudson.plugins.git.extensions.impl.WipeWorkspace')
                        }
                    }
                }
            }
        }
    """
}}
