// Openshift project
OPENSHIFT_SERVICE_ACCOUNT = 'jenkins'
DOCKER_REPO_URL = 'docker-registry.default.svc.cluster.local:5000'
CI_NAMESPACE= env.CI_NAMESPACE ?: 'ai-coe'
CI_TEST_NAMESPACE = env.CI_THOTH_TEST_NAMESPACE ?: 'ai-coe'

// Defaults for SCM operations
env.ghprbGhRepository = env.ghprbGhRepository ?: 'goern/janusgraph-openshift'
env.ghprbActualCommit = env.ghprbActualCommit ?: 'master'

// If this PR does not include an image change, then use this tag
STABLE_LABEL = "stable"
tagMap = [:]

// Initialize
tagMap['janusgraph'] = '0.2.0'

// IRC properties
IRC_NICK = "aicoe-bot"
IRC_CHANNEL = "#thoth-station"

properties(
    [
        buildDiscarder(logRotator(artifactDaysToKeepStr: '30', artifactNumToKeepStr: '', daysToKeepStr: '90', numToKeepStr: '')),
        disableConcurrentBuilds(),
    ]
)


library(identifier: "cico-pipeline-library@master",
        retriever: modernSCM([$class: 'GitSCMSource',
                              remote: "https://github.com/CentOS/cico-pipeline-library",
                              traits: [[$class: 'jenkins.plugins.git.traits.BranchDiscoveryTrait'],
                                       [$class: 'RefSpecsSCMSourceTrait',
                                        templates: [[value: '+refs/heads/*:refs/remotes/@{remote}/*']]]]])
                            )
library(identifier: "ci-pipeline@master",
        retriever: modernSCM([$class: 'GitSCMSource',
                              remote: "https://github.com/CentOS-PaaS-SIG/ci-pipeline",
                              traits: [[$class: 'jenkins.plugins.git.traits.BranchDiscoveryTrait'],
                                       [$class: 'RefSpecsSCMSourceTrait',
                                        templates: [[value: '+refs/heads/*:refs/remotes/@{remote}/*']]]]])
                            )
library(identifier: "ai-stacks-pipeline@master",
        retriever: modernSCM([$class: 'GitSCMSource',
                              remote: "https://github.com/goern/AI-Stacks-pipeline",
                              traits: [[$class: 'jenkins.plugins.git.traits.BranchDiscoveryTrait'],
                                       [$class: 'RefSpecsSCMSourceTrait',
                                        templates: [[value: '+refs/heads/*:refs/remotes/@{remote}/*']]]]])
                            )

pipeline {
    agent {
        kubernetes {
            cloud 'openshift'
            label 'ai-stacks-pipeline-' + env.ghprbActualCommit
            serviceAccount OPENSHIFT_SERVICE_ACCOUNT
            containerTemplate {
                name 'jnlp'
                args '${computer.jnlpmac} ${computer.name}'
                image DOCKER_REPO_URL + '/'+ CI_NAMESPACE +'/jenkins-aicoe-slave:' + STABLE_LABEL
                ttyEnabled false
                command ''
            }
        }
    }
    stages {
        stage("Setup Build Templates") {
            steps {
                script {
                    aIStacksPipelineUtils.createBuildConfigs(CI_TEST_NAMESPACE)
                }
            }
        }
        stage("Get Changelog") {
            steps {
                node('master') {
                    script {
                        env.changeLogStr = pipelineUtils.getChangeLogFromCurrentBuild()
                        echo env.changeLogStr
                    }
                    writeFile file: 'changelog.txt', text: env.changeLogStr
                    archiveArtifacts allowEmptyArchive: true, artifacts: 'changelog.txt'
                }
            }
        }
        stage("Build Container Images") {
            steps {
                echo "Building JanusGraph container image..."
                script {
                    tagMap['janusgraph'] = aIStacksPipelineUtils.buildImageWithTag(CI_TEST_NAMESPACE, "janusgraph", '0.2.0')
                }
            }
        }
        stage("Image Tag Report") {
            steps {
                script {
                    pipelineUtils.printLabelMap(tagMap)
                }
            }
        }
    }
    post {
        always {
            script {
                String prMsg = ""
                if (env.ghprbActualCommit != null && env.ghprbActualCommit != "master") {
                    prMsg = "(PR #${env.ghprbPullId} ${env.ghprbPullAuthorLogin})"
                }
                def message = "${JOB_NAME} ${prMsg} build #${BUILD_NUMBER}: ${currentBuild.currentResult}: ${BUILD_URL}"

                pipelineUtils.sendIRCNotification("${IRC_NICK}", IRC_CHANNEL, message)
                mattermostSend channel: "#thoth-station", icon: 'https://avatars1.githubusercontent.com/u/33906690', message: "${message}"
            }
        }
        success {
            echo "All Systems GO!"
        }
        failure {
            error "BREAK BREAK BREAK - build failed!"
        }
    }
}
