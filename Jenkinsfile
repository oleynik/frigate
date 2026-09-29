// Builds Frigate from source out of this fork and pushes it to the homelab
// Nexus -- same shape and same destination as the patched neolink image, so
// the job's Script Path stays at the default.
//
// This fork exists to carry one patch: docker/main/install_deps.sh installs
// mesa-va-drivers from bookworm, which predates the gfx1151 iGPU on the NVR
// host, so VAAPI cannot initialise. Redmine #113 and, in the homelab-setup
// repo, docs/nvr-hwaccel.md.
//
// Unlike neolink this is a genuine from-source build of the whole image:
// expect roughly 90 minutes, not a pull. The timeout is set accordingly.
pipeline {
    agent { label 'docker' }

    environment {
        REGISTRY = 'docker-hosted.nexus.homelab.lan'
        IMAGE    = 'frigate'
    }

    options {
        // Well above neolink's 90 minutes: this compiles the image rather than
        // layering onto a released one, and a cold agent with no build cache
        // is the slow case that has to fit.
        timeout(time: 4, unit: 'HOURS')
        disableConcurrentBuilds()
    }

    stages {
        stage('Tag') {
            steps {
                script {
                    // Generate frigate/version.py and web/.env. The build FAILS
                    // without them, so this is a build step, not a convenience
                    // -- and it must run before the tag is read. It stands in
                    // for `make version` because the agent has no make, so it
                    // must mirror the Makefile's `version` target byte for byte
                    // (same git command, same two lines); recheck it whenever a
                    // rebase touches that target. VERSION is read from the
                    // Makefile so upstream bumps flow through, and a line that
                    // does not parse stops the build rather than tagging an
                    // image with an empty or mangled version.
                    sh '''
                        set -eu
                        VERSION=$(sed -n 's/^VERSION[[:space:]]*=[[:space:]]*//p' Makefile)
                        case "$VERSION" in
                            ''|*[!A-Za-z0-9_.-]*)
                                echo "No parseable VERSION line in the Makefile (got: '$VERSION')" >&2
                                exit 1 ;;
                        esac
                        COMMIT_HASH=$(git log -1 --pretty=format:%h)
                        printf 'VERSION = "%s-%s"\\n' "$VERSION" "$COMMIT_HASH" > frigate/version.py
                        printf 'VITE_GIT_COMMIT_HASH=%s\\n' "$COMMIT_HASH" > web/.env
                    '''
                    // Take the tag from version.py rather than computing one:
                    // the step above writes "<version>-<short-sha>" there, and
                    // that same string is what the running Frigate reports in
                    // its UI and /api/version. A tag derived any other way can
                    // disagree with what the container says it is.
                    env.FRIGATE_VERSION = sh(
                        script: '''sed -n 's/^VERSION = "\\(.*\\)"$/\\1/p' frigate/version.py''',
                        returnStdout: true).trim()
                    env.TAG = "${REGISTRY}/${IMAGE}:${env.FRIGATE_VERSION}"
                }
                echo "Building ${env.TAG} from ${env.BRANCH_NAME}"
            }
        }

        stage('Build') {
            steps {
                // The same command the Makefile's amd64 target runs, plus
                // --load so the image lands in the local image store for the
                // push and the digest inspect below. Only amd64 is built --
                // there is no arm64 host here and it would double the runtime.
                sh '''
                    set -eu
                    docker buildx build --target=frigate \
                        --file docker/main/Dockerfile . \
                        --platform linux/amd64 \
                        --tag "$TAG" \
                        --load
                '''
            }
        }

        stage('Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-credentials',
                                                  usernameVariable: 'NEXUS_USER',
                                                  passwordVariable: 'NEXUS_PASS')]) {
                    sh '''
                        set -eu
                        printf '%s' "$NEXUS_PASS" | docker login "$REGISTRY" -u "$NEXUS_USER" --password-stdin
                        docker push "$TAG"
                        echo "DIGEST: $(docker inspect --format '{{index .RepoDigests 0}}' "$TAG")"
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout "$REGISTRY" || true'
            // TAG is unset when the Tag stage never got that far.
            sh '[ -z "${TAG:-}" ] || docker image rm -f "$TAG" || true'
        }
    }
}
