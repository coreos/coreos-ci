// Useful references:
// - https://pagure.io/fedora-qa/fedora_openqa/blob/main/f/src/fedora_openqa/consumer.py
// - https://pagure.io/fedora-ci/general/issue/436
// - https://github.com/json-path/JsonPath
// - https://sumiya.page/jpath.html

// For now, we literally list all the RPMs to trigger on in the JMS selector
// below. This doesn't scale though. In the future, we'll either use e.g.
// `critpath_groups` or have a separate listener that triggers this job.
def srpms
def testlist

node {
    checkout scm
    def testlist_raw = readYaml(file: "bodhi-testing.yaml")
    // convert to a dict of { name -> object }
    testlist = (testlist_raw["gated-srpms"] + testlist_raw["srpms"]).collectEntries {[it["name"], it]}
    srpms = testlist.keySet() as List

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
}

properties([
  pipelineTriggers([
    ciBuildTrigger(
      noSquash: true,
      // Note a core principle here is that we're trying to push as much
      // filtering as possible into the `checks` so that a job is only
      // actually run when we need to test something. This is why we have those
      // gnarly `update.builds[].nvr` checks below.
      providerList: [
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
                [field: '$.update.release.name', expectedValue: "^F(${releasevers.join("|")})\$"],
                // Is this for an SRPM we care about? The jsonpath expression
                // here evaluates to "all builds of type rpm whose nvr matches
                // 'n-v-r' where 'n' is one of those in $srpms".
                [field: "\$.update.builds[?(@.type == 'rpm' && @.nvr =~ /^(${srpms.join("|")})-[^-]+-[^-]+\$/)]", expectedValue: '^\\[.+\\]$']
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
        // Let's not try to be fancy; if the update has multiple bundled builds,
        // just run all the tests. That shouldn't happen very often anyway, so it's not
        // worth the complexity of trying to combine.
        def test_patterns = ""
        def testiso_patterns = ""
        if (msg.update.builds.size() == 1) {
            def srpm = msg.update.builds[0].nvr.split('-')[0..-3].join('-')
            test_patterns += testlist[srpm]['test-patterns'] ?: ""
            testiso_patterns += testlist[srpm]['testiso-patterns'] ?: ""
        }

        // always force at least `basic` to run; that way even if all of the
        // specified tests are denylisted because they're temporarily broken, we
        // still get a sanity-check that it doesn't break boot
        if (test_patterns != "" && test_patterns != "skip") {
            test_patterns += " basic"
        }

        test = build(job: 'test-override', propagate: false, wait: true,
                     parameters: [
                        string(name: 'STREAM', value: stream),
                        string(name: 'OVERRIDES', value: msg.update.url),
                        string(name: 'TESTS', value: test_patterns),
                        string(name: 'TESTISO_TESTS', value: testiso_patterns),
                     ])
    }

    stage("Report Completion") {
        def outcome
        // treat UNSTABLE as PASSED too; we often have expired snoozed tests
        // that'll warn and Greenwave/Bodhi treats NEEDS_INSPECTION outcomes
        // as blocking
        if (test.result == 'SUCCESS' || test.result == 'UNSTABLE') {
            outcome = 'PASSED'
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
