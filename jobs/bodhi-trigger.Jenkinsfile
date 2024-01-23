// Useful references:
// - https://pagure.io/fedora-qa/fedora_openqa/blob/main/f/src/fedora_openqa/consumer.py
// - https://pagure.io/fedora-ci/general/issue/436
// - https://github.com/json-path/JsonPath
// - https://sumiya.page/jpath.html

// XXX: we should move this list to a separate YAML file for knob extensibility
// in the future (for e.g. which tests to run for each)
// XXX: one idea is to trigger for all packages in FCOS and only run `basic` by
// default, and only run more tests for a given subset like the below
def srpms = [
    'container-selinux',
    'glibc',
    'grub2',
    'ignition',
    'kernel',
    'kexec-tools',
    'NetworkManager',
    'ostree',
    'podman',
    'rpm-ostree',
    'rust-afterburn',
    'rust-coreos-installer',
    'rust-zincati',
    'selinux-policy',
    'systemd'
]

node {
    // XXX: we should drain what we need of pipeutils into coreos-ci-lib
    shwrap("rm -rf pipe && git clone https://github.com/coreos/fedora-coreos-pipeline --depth=1 pipe")
    // these are script global vars
    pipeutils = load("pipe/utils.groovy")
    // XXX: should use load_pipecfg but it checks for arch availability
    pipecfg = readYaml(file: "pipe/config.yaml")

    // query the currently active streams and collect their releasevers; we'll
    // use these to update our triggers
    streams = pipeutils.streams_of_type(pipecfg, 'development') +
              pipeutils.streams_of_type(pipecfg, 'mechanical')
    stream_to_releasever = [:]
    for (st in streams) {
        // not particularly proud of this, but... it'll work for now
        shwrap("curl -sSLO https://raw.githubusercontent.com/coreos/fedora-coreos-config/${st}/manifest.yaml")
        def yaml = readYaml(file: "manifest.yaml")
        stream_to_releasever[st] = yaml.releasever
    }

    releasevers = stream_to_releasever.values() as Set

    // query which ones use updates-testing; for more context, see
    // https://github.com/coreos/coreos-ci/pull/49#issuecomment-1769679589
    bodhi_composed_releasevers = [] as Set
    pungi_composed_releasevers = [] as Set
    for (releasever in releasevers) {
        def url = "https://bodhi.fedoraproject.org/releases/?name=F${releasever}"
        if (shwrapRc("curl -sSL ${url} | jq -e '.releases[0].composed_by_bodhi'") == 0) {
            bodhi_composed_releasevers += releasever
        } else {
            pungi_composed_releasevers += releasever
        }
    }
}

properties([
  pipelineTriggers([
    ciBuildTrigger(
      noSquash: true,
      // The list of triggers below is messy; a core principle here is that
      // we're trying to push as much filtering as possible into the `checks`
      // so that a job is only actually run when we need to test something.
      // This is why we have those gnarly `update.builds[].nvr` checks below.
      // We should be able to simplify this list a lot once we have
      // https://pagure.io/fedora-ci/general/issue/436.
      providerList: [
        // TL;DR: Effectively triggered when a rawhide/branched update was opened.
        // Longer explanation: Triggered when an update was pushed to updates-
        // testing. Since for rawhide (and at the start, branched) updates-
        // testing doesn't exist, it's immediately triggered on update creation.
        // We don't care about the non-rawhide case since we don't want to test
        // when the update is pushed to updates-testing but right away when it's
        // created (covered by the next trigger below).
        rabbitMQSubscriber(
            name: 'Fedora Messaging',
            overrides: [
                topic: 'org.fedoraproject.prod.bodhi.update.status.testing.koji-build-group.build.complete',
                // https://github.com/jenkinsci/jms-messaging-plugin/issues/262
                queue: 'eb65f4a0-9daa-4689-9bdb-910158b3abba'
            ],
            checks: [
                [field: '$.update.release.id_prefix', expectedValue: '^FEDORA$'],
                // Is this for a release we care about?
                [field: '$.update.release.name', expectedValue: "^F(${pungi_composed_releasevers.join("|")})\$"],
                // Is this for an SRPM we care about? The jsonpath expression
                // here evaluates to "all builds of type rpm whose nvr matches
                // 'n-v-r' where 'n' is one of those in $srpms".
                [field: "\$.update.builds[?(@.type == 'rpm' && @.nvr =~ /^(${srpms.join("|")})-[^-]+-[^-]+\$/)]", expectedValue: '^\\[.+\\]$'],
                // We handle the retrigger case separately below.
                [field: '$.re-trigger', expectedValue: '^false$']
            ]
        ),
        // TL;DR: Effectively triggered when a non-rawhide update was opened.
        // Longer explanation: Triggered when an update was requested to be
        // pushed to updates-testing. This never happens for rawhide, and for
        // non-rawhide usually happens shortly after the update is opened.
        rabbitMQSubscriber(
            name: 'Fedora Messaging',
            overrides: [
                topic: 'org.fedoraproject.prod.bodhi.update.request.testing',
                queue: '6455c3a4-9194-4cc0-a7f6-79864036ded2'
            ],
            checks: [
                [field: '$.update.release.id_prefix', expectedValue: '^FEDORA$'],
                [field: '$.update.release.name', expectedValue: "^F(${bodhi_composed_releasevers.join("|")})\$"],
                [field: "\$.update.builds[?(@.type == 'rpm' && @.nvr =~ /^(${srpms.join("|")})-[^-]+-[^-]+\$/)]", expectedValue: '^\\[.+\\]$']
            ]
        ),
        // Triggered when an update's tests are requested to be rerun.
        rabbitMQSubscriber(
            name: 'Fedora Messaging',
            overrides: [
                topic: 'org.fedoraproject.prod.bodhi.update.status.testing.koji-build-group.build.complete',
                queue: '036407bc-20ad-4290-bdee-deeab6ff6d7a'
            ],
            checks: [
                [field: '$.update.release.id_prefix', expectedValue: '^FEDORA$'],
                [field: '$.update.release.name', expectedValue: "^F(${releasevers.join("|")})\$"],
                [field: "\$.update.builds[?(@.type == 'rpm' && @.nvr =~ /^(${srpms.join("|")})-[^-]+-[^-]+\$/)]", expectedValue: '^\\[.+\\]$'],
                // Were we asked to re-run the tests?
                [field: '$.re-trigger', expectedValue: '^true$']
            ]
        ),
        // Triggered when an update is edited. We only care about the case when
        // the edit changes the builds in the update (and not e.g. changing
        // the description).
        rabbitMQSubscriber(
            name: 'Fedora Messaging',
            overrides: [
                topic: 'org.fedoraproject.prod.bodhi.update.edit',
                queue: '9a5c5403-0d3f-403c-81b0-e0a6414513b2'
            ],
            checks: [
                [field: '$.update.release.id_prefix', expectedValue: '^FEDORA$'],
                [field: '$.update.release.name', expectedValue: "^F(${releasevers.join("|")})\$"],
                [field: "\$.update.builds[?(@.type == 'rpm' && @.nvr =~ /^(${srpms.join("|")})-[^-]+-[^-]+\$/)]", expectedValue: '^\\[.+\\]$'],
                // Were any builds actually changed?
                [field: 'concat($.new_builds,$.removed_builds)', expectedValue: '^.*$'],
            ]
        )
      ]
    )
  ]),
])

def test_mode = false
def raw_message = env.CI_MESSAGE
if (raw_message == null) {
    raw_message = input(message: 'Triggered manually; running in test mode.', parameters: [text('CI_MESSAGE')])
    test_mode = true
}

def msg = readJSON(text: raw_message)
println("Handling message: ${msg}")

// collect streams with matching releasever
def releasever = msg.update.release.version.toInteger()
def matching_streams = []
for (entry in stream_to_releasever) {
    if (entry.value == releasever) {
        matching_streams += [entry.key]
    }
}

// This should be covered by the checks embedded in the triggers above, but
// because of the classic "delayed by one" relationship with the job being
// triggered updating its own triggers, manually check for this here too.
if (matching_streams.isEmpty()) {
    println("Update is for f${releasever} which we don't track anymore.")
    return
}

def stream = ""
if (matching_streams.size() == 1) {
    stream = matching_streams[0]
} else if ("testing-devel" in matching_streams) {
    // It's possible multiple streams track the same version. By far, the most
    // likely case is testing-devel and next-devel. In the future, we could trigger
    // for both, but for now just trigger for testing-devel.
    stream = "testing-devel"
} else if ("next-devel" in matching_streams) {
    // Our second-favourite choice in a tie.
    stream = "next-devel"
} else {
    // that shouldn't ever happen, so let's error out for now and if we hit it,
    // we'll figure out what to do
    error("multiple non-devel matching streams found: ${matching_streams}")
}

currentBuild.description = msg.update.builds[0].nvr

if (test_mode) {
    println("Would trigger test-override with STREAM=${stream}")
    return
}

cosaPod(cpu: "0.1", kvm: false) {
    def test = null

    stage("Report Running") {
        withCredentials([usernamePassword(credentialsId: 'resultsdb-auth',
                                          usernameVariable: 'RDB_USERNAME',
                                          passwordVariable: 'RDB_PASSWORD')]) {
            shwrap("""/usr/lib/coreos-assembler/resultsdb-report \
                --testcase cosa.build-and-test \
                --testcase-url ${JENKINS_URL}/job/test-override \
                --testrun-url ${JENKINS_URL}/job/test-override \
                --outcome RUNNING --advisory ${msg.update.updateid} \
                --stream ${stream}
            """)
        }
    }

    stage("Test") {
        test = build(job: 'test-override', propagate: false, wait: true,
                     parameters: [
                        string(name: 'STREAM', value: stream),
                        string(name: 'OVERRIDES', value: msg.update.url),
                     ])
    }

    stage("Report Completion") {
        def outcome
        if (test.result == 'SUCCESS') {
            outcome = 'PASSED'
        } else if (test.result == 'UNSTABLE') {
            outcome = 'NEEDS_INSPECTION'
        } else {
            outcome = 'FAILED'
        }
        def blueocean_url = "${JENKINS_URL}/blue/organizations/jenkins/test-override/detail/test-override/${test.number}"
        withCredentials([usernamePassword(credentialsId: 'resultsdb-auth',
                                          usernameVariable: 'RDB_USERNAME',
                                          passwordVariable: 'RDB_PASSWORD')]) {
            shwrap("""/usr/lib/coreos-assembler/resultsdb-report \
                --testcase cosa.build-and-test \
                --testcase-url ${JENKINS_URL}/job/test-override \
                --testrun-url ${blueocean_url} \
                --outcome ${outcome} --advisory ${msg.update.updateid} \
                --stream ${stream}
            """)
        }
        if (test.result != 'SUCCESS') {
            matrixSendMessage(hostname: 'fedora.im', accessTokenCredentialsId: 'CREDS', roomId: 'ROOM', body: "${currentBuild.description} - ${outcome}")
        }
    }

    // propagate
    currentBuild.result = test.result
}
