gerritReview labels: [Verified: 0], message: "Test started: ${env.BUILD_URL}"
@Library('sw-jenkins-library@b303b82fe823acd2ef8d0f77498e87a4773aa8ea') _

pipeline {
    agent {
        label 'exploration_tool'
    }

    stages {
        stage('Setup') {
            steps {
                sh 'git clean -xdf'
            }
        }
        stage('Offline tests') {
            agent {
                dockerfile {
                    reuseNode true
                }
            }
            steps {
                sh 'tox'
            }
        }
        stage('XM112 integration tests') {
            options {
                lock resource: '${env.NODE_NAME}-xm112'
            }
            stages {
                stage('Flash') {
                    steps {
                        findBuildAndCopyArtifacts(projectName: 'sw-main', revision: "master",
                                                  artifactNames: ["internal_stash_python_libs.tgz", "internal_stash_binaries_xm112.tgz"])
                        sh 'rm -rf stash && mkdir stash'
                        sh 'tar -xzf internal_stash_python_libs.tgz -C stash'
                        sh 'tar -xzf internal_stash_binaries_xm112.tgz -C stash'
                        sh '(cd stash && python3 python_libs/test_utils/flash.py)'
                    }
                }
                stage('Integration tests') {
                    agent {
                        dockerfile {
                            reuseNode true
                            args '--net=host --privileged'
                        }
                    }
                    steps {
                        sh 'python3 -m pip install -q -U --user ".[test]"'
                        sh 'python3 -m pytest -p no:pytest-qt -v tests/integration --uart --spi'
                    }
                }
            }
        }
        stage('Exploration server integration tests')
        {
            options {
                lock resource: '${env.NODE_NAME}-localhost'
            }
            stages {
                stage('Retrieve stash') {
                    steps {
                        findBuildAndCopyArtifacts(projectName: 'sw-main', revision: "master",
                                                artifactNames: ["internal_stash_binaries_sanitizer_a111.tgz"])
                        sh 'rm -rf stash && mkdir stash'
                        sh 'tar -xzf internal_stash_binaries_sanitizer_a111.tgz -C stash'
                    }
                }
                stage('Integration test') {
                    agent {
                        dockerfile {
                            reuseNode true
                        }
                    }
                    steps {
                        sh 'python3 -m pip install -q -U --user ".[test]"'
                        sh 'tests/run-integration-tests.sh'
                    }
                }
            }
        }
        stage('GUI tests') {
            agent {
                dockerfile {
                    reuseNode true
                }
            }
            steps {
                sh 'python3 -m pip install -U --user ".[test,app]"'
                sh 'python3 -m pytest -v --timeout=60 --timeout_method=thread tests/gui'
            }
        }
    }

    post {
        success { gerritReview labels: [Verified: 1], message: "Success: ${env.BUILD_URL}" }
        failure { gerritReview labels: [Verified: -1], message: "Failed: ${env.BUILD_URL}" }
        aborted { gerritReview labels: [Verified: -1], message: "Aborted: ${env.BUILD_URL}" }
    }
}
