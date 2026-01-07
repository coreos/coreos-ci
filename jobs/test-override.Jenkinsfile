node {
    // XXX: we should drain what we need of pipeutils into coreos-ci-lib
    shwrap("rm -rf pipe && git clone https://github.com/coreos/fedora-coreos-pipeline --depth=1 pipe")
    // these are script global vars
    pipeutils = load("pipe/utils.groovy")
    // XXX: should use load_pipecfg but it checks for arch availability
    pipecfg = readYaml(file: "pipe/config.yaml")
}

properties([
    parameters([
        string(name: 'OVERRIDES',
               defaultValue: "",
               description: '''Space-separated list of Bodhi or Koji Build or Koji Task (scratch) URLs or COPR Build
For example:
- The Bodhi Build URL is https://bodhi.fedoraproject.org/updates/FEDORA-2025-16cced9e85
- The Koji Build URL is https://koji.fedoraproject.org/koji/buildinfo?buildID=2681652
- The Koji Task (scratch) URL is https://koji.fedoraproject.org/koji/taskinfo?taskID=132021551
- To support COPR builds from any COPR repo, the COPR Build is in the format: copr/<build-id>'''),
        choice(name: 'STREAM',
               choices: pipeutils.streams_of_type(pipecfg, 'development') +
                        pipeutils.streams_of_type(pipecfg, 'mechanical'),
               description: 'CoreOS development stream to test'),
        string(name: 'SKIP_TESTS_ARCHES',
               description: 'Space-separated list of architectures to skip tests on',
               defaultValue: "",
               trim: true),
        string(name: 'COREOS_ASSEMBLER_IMAGE',
               description: 'Override coreos-assembler image to use',
               defaultValue: "",
               trim: true),
        string(name: 'TESTS',
               description: 'Space-separated test patterns to use (or "skip" to skip)',
               defaultValue: "",
               trim: true),
        string(name: 'TESTISO_TESTS',
               description: 'Space-separated testiso test patterns to use (or "skip" to skip)',
               defaultValue: "",
               trim: true),
        string(name: 'DESCRIPTION',
               description: 'Description to use when reporting results in Jenkins and Matrix',
               defaultValue: "",
               trim: true),
        booleanParam(name: 'ALLOW_KOLA_UPGRADE_FAILURE',
                     defaultValue: false,
                     description: "Don't error out if upgrade tests fail (temporary)"),
        booleanParam(name: 'REPORT_TO_RESULTSDB',
                     defaultValue: false,
                     description: "Report results to resultsdb for the Bodhi update"),
    ]),
    buildDiscarder(logRotator(
        numToKeepStr: '100',
        artifactNumToKeepStr: '30'
    )),
    durabilityHint('PERFORMANCE_OPTIMIZED')
])

// If the user provided a description we'll start with that. If it's
// "" then we'll populate it below.
def descPrefix = params.DESCRIPTION

def blueocean_url = "${env.RUN_DISPLAY_URL}"

def bodhi_update_ids = []
def koji_build_ids = []
def koji_scratch_build_ids = []
def copr_build_ids = []

def bodhi_url_prefix = "https://bodhi.fedoraproject.org/updates/"
def koji_url_prefix = "https://koji.fedoraproject.org/koji/buildinfo?buildID="
def koji_scratch_url_prefix = "https://koji.fedoraproject.org/koji/taskinfo?taskID="
// To support builds from any COPR repo and not just the CoreOS continuous repo
// check if the prefix is copr/<build-id>
def copr_build_prefix = "copr/"

for (url in params.OVERRIDES.split()) {
    if (url.startsWith(bodhi_url_prefix)) {
        bodhi_update_ids += url - bodhi_url_prefix
    } else if (url.startsWith(koji_url_prefix)) {
        koji_build_ids += url - koji_url_prefix
    } else if (url.startsWith(koji_scratch_url_prefix)) {
        koji_scratch_build_ids += url - koji_scratch_url_prefix
    } else if (url.startsWith(copr_build_prefix)) {
        copr_build_ids += url - copr_build_prefix
    } else {
        error("don't know how to handle override URL $url")
    }
}

if (bodhi_update_ids.size() == 0 && koji_build_ids.size() == 0 && koji_scratch_build_ids.size() == 0 && copr_build_ids.size() == 0) {
    error("no overrides provided")
}

if (params.REPORT_TO_RESULTSDB && bodhi_update_ids.size() != 1) {
    error("When requesting resultsdb reporting can only provide one Bodhi URL")
}

// runtime parameter always wins
def cosa_img = params.COREOS_ASSEMBLER_IMAGE
cosa_img = cosa_img ?: pipeutils.get_cosa_img(pipecfg, params.STREAM)
def stream_info = pipecfg.streams[params.STREAM]

// Grab any environment variables we should set
def container_env = pipeutils.get_env_vars_for_stream(pipecfg, params.STREAM)

// Keep in sync with build.Jenkinsfile
def cosa_memory_request_mb = 10.5 * 1024 as Integer
def ncpus = ((cosa_memory_request_mb - 512) / 1536) as Integer

currentBuild.description = "[${descPrefix}] Pending"


cosaPod(image: cosa_img,
        cpu: "${ncpus}", memory: "${cosa_memory_request_mb}Mi",
        env: container_env,
        serviceAccount: "jenkins") {
timeout(time: 200, unit: 'MINUTES') {
try {
    // Set a sane build description if none was provided
    if (descPrefix == "") {
        if (!bodhi_update_ids.isEmpty()) {
            descPrefix = shwrapCapture("curl -sSL https://bodhi.fedoraproject.org/updates/${bodhi_update_ids[0]} | jq -r .update.title")
        } else if (!koji_build_ids.isEmpty()) {
            descPrefix = koji_build_ids[0]
        } else if (!koji_scratch_build_ids.isEmpty()) {
            descPrefix = shwrapCapture("""
                koji taskinfo -v -r ${koji_scratch_build_ids[0]} | \
                grep SRPM: | head -1 | awk -F '/' '{print \$NF}' | sed 's#.src.rpm\$##'
            """)
            descPrefix = "scratch: ${descPrefix}"
        } else {
            descPrefix = shwrapCapture("""
                curl -sSL 'https://copr.fedorainfracloud.org/api_3/build/${copr_build_ids[0]}' | \
                jq -r '.source_package | \"\\(.name)-\\(.version)\"'
            """)
            descPrefix = "copr: ${descPrefix}"
        }
    }
    currentBuild.description = "[${descPrefix}] Running"

    // resolve bodhi IDs to koji builds
    for (id in bodhi_update_ids) {
        def build_ids = shwrapCapture("curl -sSL https://bodhi.fedoraproject.org/updates/${id} | jq -r .update.builds[].nvr").split() as List
        if (build_ids.size() == 0) {
            error("Bodhi update ${id} has no builds!")
        }
        koji_build_ids += build_ids
    }

    if (params.REPORT_TO_RESULTSDB) {
        stage("Report Running") {
            withCredentials([usernamePassword(credentialsId: 'resultsdb-auth',
                                              usernameVariable: 'RDB_USERNAME',
                                              passwordVariable: 'RDB_PASSWORD')]) {
                shwrap("""/usr/lib/coreos-assembler/resultsdb-report    \
                    --testcase cosa.build-and-test                      \
                    --testcase-url ${JENKINS_URL}/job/test-override     \
                    --testrun-url ${blueocean_url}                      \
                    --outcome RUNNING --advisory ${bodhi_update_ids[0]} \
                    --stream ${params.STREAM}
                """)
            }
        }
    }

    def arches = pipeutils.get_additional_arches(pipecfg, params.STREAM) as Set
    // XXX: this should be further intersected with the set of arches relevant
    // for the test subject
    arches = arches.intersect(pipeutils.get_supported_additional_arches()).plus("x86_64")

    // Mechanical streams are unlocked; in this case, we want to base our build
    // on the latest known working build so that the only unknown is the test
    // subject. Do this unconditionally for now, we can add a param if we want
    // to be able to disable this heuristic.
    def build_lock = ""
    if (stream_info.type == "mechanical") {
        def builds_raw = shwrapCapture("curl -sSL https://builds.coreos.fedoraproject.org/prod/streams/${params.STREAM}/builds/builds.json")
        def builds = readJSON text: builds_raw
        // find newest build with all the arches concerned passing
        for (build in builds["builds"]) {
            if (build.arches.containsAll(arches)) {
                build_lock = build.id
                break
            }
        }
    }

    def (url, branch) = pipeutils.get_source_config_for_stream(pipecfg, params.STREAM)
    def src_config_commit = ""
    if (build_lock == "") {
        src_config_commit = shwrapCapture("""
            git ls-remote ${url} refs/heads/${branch} | cut -d \$'\t' -f 1
        """)
    } else {
        def meta_raw = shwrapCapture("curl -sSL https://builds.coreos.fedoraproject.org/prod/streams/${params.STREAM}/builds/${build_lock}/x86_64/meta.json")
        def meta = readJSON text: meta_raw
        src_config_commit = meta["coreos-assembler.container-config-git"]["commit"]
    }
    shwrap("cosa init --branch ${branch} --commit=${src_config_commit} ${url}")

    // Initialize the sessions on the remote builders
    def archinfo = arches.collectEntries{[it, [session: ""]]}
    stage("Initialize Remotes") {
        parallel archinfo.keySet().collectEntries{arch -> [arch, {
            if (arch != "x86_64") {
                pipeutils.withPodmanRemoteArchBuilder(arch: arch) {
                    archinfo[arch]['session'] = shwrapCapture("""
                    cosa remote-session create --image ${cosa_img} --expiration 4h --workdir ${env.WORKSPACE}
                    """)
                    withEnv(["COREOS_ASSEMBLER_REMOTE_SESSION=${archinfo[arch]['session']}"]) {
                        shwrap("""
                        cosa init --branch ${branch} --commit=${src_config_commit} ${url}
                        """)
                    }
                }
            }
            // buildfetch (allows for showing RPM diff)
            pipeutils.withOptionalExistingCosaRemoteSession(arch: arch, session: archinfo[arch]['session']) {
                def buildfetch_args = ""
                if (build_lock != "") {
                    // fetch the build we'll lock from and the associated lockfile
                    buildfetch_args = "--build ${build_lock} --file manifest-lock.generated.${arch}.json"
                }
                shwrap("cosa buildfetch --stream ${params.STREAM} ${buildfetch_args}")
            }
        }]}
    }

    // Download overrides
    stage("Download Overrides") {
        def parallelruns = [:]
        for (architecture in archinfo.keySet()) {
            def arch = architecture
            parallelruns[arch] = {
                pipeutils.withOptionalExistingCosaRemoteSession(arch: arch, session: archinfo[arch]['session']) {
                    for (id in koji_build_ids) {
                        // XXX: I think this will fail for a package that doesn't have noarch
                        // packages nor packages on $arch
                        shwrap("cosa shell -- env -C overrides/rpm koji download-build ${id} --arch ${arch} --arch noarch")
                    }
                    // Download Koji Task (scratch) rpms
                    for (id in koji_scratch_build_ids) {
                        shwrap("cosa shell -- env -C overrides/rpm koji download-task ${id} --arch ${arch} --arch noarch --skip='debug|test'")
                    }
                    // Download copr build rpms
                    for (id in copr_build_ids) {
                        // get chrootname like fedora-42-x86_64
                        def manifest_raw = shwrapCapture("cosa shell -- env -C src/config cat manifest.yaml")
                        def manifest_yaml = readYaml text: manifest_raw
                        def chrootname = ""
                        if (manifest_yaml.releasever != "") {
                            chrootname = "fedora-${manifest_yaml.releasever}-${arch}"
                        } else {
                            error("can not find releasever in manifest.yaml")
                        }
                        shwrap("cosa shell -- env -C overrides/rpm copr-cli download-build --rpms --chroot ${chrootname} ${id}")
                        shwrap("cosa shell -- env -C overrides/rpm rsync -av --remove-source-files --exclude='*debug*' --exclude='*test*' --exclude='*src*' ${chrootname}/ .")
                    }
                }
            }
        }
        parallel parallelruns
    }

    def skip_tests_arches = params.SKIP_TESTS_ARCHES.split()
    def parallelruns = [:]
    for (architecture in archinfo.keySet()) {
        def arch = architecture
        if (arch in skip_tests_arches) {
            // The user has explicitly told us that it is OK to
            // skip tests on this architecture. Presumably because
            // they already passed in a previous run.
            continue
        }
        parallelruns[arch] = {
            pipeutils.withOptionalExistingCosaRemoteSession(arch: arch, session: archinfo[arch]['session']) {
                def autolock_arg = ""
                def parent_arg = ""
                if (build_lock != "") {
                    autolock_arg = "--autolock ${build_lock}"
                    parent_arg = "--parent-build ${build_lock}"
                }
                stage("${arch}:Fetch") {
                    shwrap("cosa fetch --with-cosa-overrides ${autolock_arg}")
                }
                stage("${arch}:Build OS Container") {
                    shwrap("cosa build --force ${autolock_arg} ${parent_arg}")
                }
                stage("${arch}:Build QEMU") {
                    shwrap("cosa osbuild qemu")
                }
                if (params.TESTS != "skip") {
                    def n = ncpus - 1 // remove 1 for upgrade test
                    kola(cosaDir: env.WORKSPACE, parallel: n, arch: arch, extraArgs: params.TESTS,
                         marker: arch, allowUpgradeFail: params.ALLOW_KOLA_UPGRADE_FAILURE)
                }
                if (params.TESTISO_TESTS != "skip") {
                    stage("${arch}:Build Live") {
                        shwrap("cosa osbuild metal metal4k live")
                        // Test metal4k with an uncompressed image and
                        // metal with a compressed one. Limit to 4G to be
                        // good neighbours and reduce chances of getting
                        // OOMkilled.
                        shwrap("cosa shell -- env XZ_DEFAULTS=--memlimit=4G cosa compress --artifact=metal")
                    }
                    stage("${arch}:kola:testiso") {
                        kolaTestIso(cosaDir: env.WORKSPACE, arch: arch, marker: arch,
                                    extraArgs: params.TESTISO_TESTS)
                    }
                }
                // Destroy the remote sessions. We don't need them anymore
                stage("${arch}:Destroy Remote") {
                    if (arch != "x86_64") {
                        shwrap("cosa remote-session destroy")
                    }
                }
            }
        }
    }
    parallel parallelruns

    currentBuild.description = "[${descPrefix}] ✔️"
    currentBuild.result = 'SUCCESS'
} catch (e) {
    currentBuild.description = "[${descPrefix}] ❌"
    currentBuild.result = 'FAILURE'
    throw e
} finally {
    // If asked to report results to resultsdb then let's do that now.
    if (params.REPORT_TO_RESULTSDB) {
        stage("Report Completion") {
            def outcome
            def emoji
            // treat UNSTABLE as PASSED too; we often have expired snoozed tests
            // that'll warn and Greenwave/Bodhi treats NEEDS_INSPECTION outcomes
            // as blocking
            if (currentBuild.result == 'SUCCESS' || currentBuild.result == 'UNSTABLE') {
                emoji = "🟢"
                outcome = 'PASSED'
            } else {
                emoji = "🔴"
                outcome = 'FAILED'
            }
            withCredentials([usernamePassword(credentialsId: 'resultsdb-auth',
                                              usernameVariable: 'RDB_USERNAME',
                                              passwordVariable: 'RDB_PASSWORD')]) {
                shwrap("""/usr/lib/coreos-assembler/resultsdb-report       \
                    --testcase cosa.build-and-test                         \
                    --testcase-url ${JENKINS_URL}/job/test-override        \
                    --testrun-url ${blueocean_url}                         \
                    --outcome ${outcome} --advisory ${bodhi_update_ids[0]} \
                    --stream ${params.STREAM}
                """)
            }
            def bodhi_url="https://bodhi.fedoraproject.org/updates/${bodhi_update_ids[0]}"
            pipeutils.matrixSend("${emoji} ${descPrefix} - [🌊](${blueocean_url}) [🪷](${bodhi_url})")
        }
    }
}}} // finally, timeout, cosaPod, finish here
