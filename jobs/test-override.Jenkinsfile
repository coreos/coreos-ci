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
               description: 'Space-separated list of Bodhi or Koji URLs'),
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
        booleanParam(name: 'ALLOW_KOLA_UPGRADE_FAILURE',
                     defaultValue: false,
                     description: "Don't error out if upgrade tests fail (temporary)"),
    ]),
    buildDiscarder(logRotator(
        numToKeepStr: '100',
        artifactNumToKeepStr: '30'
    )),
    durabilityHint('PERFORMANCE_OPTIMIZED')
])

// we'll use the first override as our description
def descPrefix = ""

def bodhi_update_ids = []
def koji_build_ids = []
for (url in params.OVERRIDES.split()) {
    if (url.startsWith("https://bodhi.fedoraproject.org/updates/")) {
        bodhi_update_ids += url - "https://bodhi.fedoraproject.org/updates/"
        descPrefix = descPrefix ?: bodhi_update_ids[0]
    } else if (url.startsWith("https://koji.fedoraproject.org/koji/buildinfo?buildID=")) {
        koji_build_ids += url -  "https://koji.fedoraproject.org/koji/buildinfo?buildID="
        descPrefix = descPrefix ?: koji_build_ids[0]
    } else {
        error("don't know how to handle override URL $url")
    }
}

if (bodhi_update_ids.size() == 0 && koji_build_ids.size() == 0) {
    error("no overrides provided")
}

// runtime parameter always wins
def cosa_img = params.COREOS_ASSEMBLER_IMAGE
cosa_img = cosa_img ?: pipeutils.get_cosa_img(pipecfg, params.STREAM)
def stream_info = pipecfg.streams[params.STREAM]

// Keep in sync with build.Jenkinsfile
def cosa_memory_request_mb = 10.5 * 1024 as Integer
def ncpus = ((cosa_memory_request_mb - 512) / 1536) as Integer

currentBuild.description = "[${descPrefix}] Pending"

cosaPod(image: cosa_img,
        cpu: "${ncpus}", memory: "${cosa_memory_request_mb}Mi",
        serviceAccount: "jenkins") {
timeout(time: 150, unit: 'MINUTES') {
try {
    // resolve to a better title if it's Bodhi
    if (!bodhi_update_ids.isEmpty()) {
        descPrefix = shwrapCapture("curl -sSL https://bodhi.fedoraproject.org/updates/${bodhi_update_ids[0]} | jq -r .update.title")
    }
    currentBuild.description = "[${descPrefix}] Running"

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

    def branch = pipeutils.get_source_config_ref_for_stream(pipecfg, params.STREAM)
    def src_config_commit = ""
    if (build_lock == "") {
        src_config_commit = shwrapCapture("""
            git ls-remote ${pipecfg.source_config.url} refs/heads/${branch} | cut -d \$'\t' -f 1
        """)
    } else {
        def meta_raw = shwrapCapture("curl -sSL https://builds.coreos.fedoraproject.org/prod/streams/${params.STREAM}/builds/${build_lock}/x86_64/meta.json")
        def meta = readJSON text: meta_raw
        src_config_commit = meta["coreos-assembler.config-gitrev"]
    }
    shwrap("cosa init --branch ${branch} --commit=${src_config_commit} ${pipecfg.source_config.url}")

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
                        cosa init --branch ${branch} --commit=${src_config_commit} ${pipecfg.source_config.url}
                        """)
                    }
                }
            }
            if (build_lock != "") {
                // fetch the build we'll lock from
                pipeutils.withOptionalExistingCosaRemoteSession(arch: arch, session: archinfo[arch]['session']) {
                    shwrap("""
                        cosa buildfetch --stream ${params.STREAM} --build ${build_lock} --file manifest-lock.generated.${arch}.json
                    """)
                }
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
                    for (id in bodhi_update_ids) {
                        // this would be `bodhi updates download`, but cosa doesn't have it and
                        // it doesn't seem worth pulling it in just for this
                        shwrap("""cosa shell -- env -C overrides/rpm sh -c '\
                                curl -sSL https://bodhi.fedoraproject.org/updates/${id} | \
                                jq -r .update.builds[].nvr | \
                                xargs -n 1 koji download-build --arch ${arch} --arch noarch'""")
                    }
                    for (id in koji_build_ids) {
                        // XXX: I think this will fail for a package that doesn't have noarch
                        // packages nor packages on $arch
                        shwrap("cosa shell -- env -C overrides/rpm koji download-build ${id} --arch ${arch} --arch noarch")
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
                if (build_lock != "") {
                    autolock_arg = "--autolock ${build_lock}"
                }
                stage("${arch}:Fetch") {
                    shwrap("cosa fetch ${autolock_arg}")
                }
                stage("${arch}:Build") {
                    shwrap("cosa build ${autolock_arg}")
                }
                def n = ncpus - 1 // remove 1 for upgrade test
                kola(cosaDir: env.WORKSPACE, parallel: n, arch: arch,
                     marker: arch, allowUpgradeFail: params.ALLOW_KOLA_UPGRADE_FAILURE)
                stage("${arch}:Build Metal") {
                    shwrap("cosa buildextend-metal")
                    shwrap("cosa buildextend-metal4k")
                }
                stage("${arch}:Build Live") {
                    shwrap("cosa buildextend-live --fast")
                    // Test metal4k with an uncompressed image and
                    // metal with a compressed one. Limit to 4G to be
                    // good neighbours and reduce chances of getting
                    // OOMkilled.
                    shwrap("cosa shell -- env XZ_DEFAULTS=--memlimit=4G cosa compress --artifact=metal")
                }
                stage("${arch}:kola:testiso") {
                    kolaTestIso(cosaDir: env.WORKSPACE, arch: arch, marker: arch)
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
}}} // cosaPod, timeout, and try finish here
