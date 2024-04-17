
def getShortCommitHash() {
    return sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
}

def getImageName() {
    return sh(returnStdout: true, script: 'make print-IMAGE_NAME').trim()
}

pipeline {
    triggers {
        githubPush()
    }
    options {
        timeout time:15, unit:'MINUTES'
        disableConcurrentBuilds()
        buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '25', numToKeepStr: '10'))
    }

    // agent { label 'ssh-slave'}
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  nodeSelector:
                    nodegroup.type: ci-workloads
                  securityContext:
                    runAsUser: 0
                  containers:
                    - name: buildah
                      image: quay.io/containers/buildah:v1.32.0
                      securityContext:
                        privileged: true
                      command:
                        - /bin/sh
                      tty: true
            '''
        }
    }

    environment {
        VERSION = 'v1.16'
        COMPLETE_VERSION = "${VERSION}.5"

        SHORT_COMMIT = getShortCommitHash()
        // IMAGE_NAME = getImageName()

        NEXUS_DOCKER_CREDS = credentials('nexus_docker_registry_creds')
        NEXUS_DOCKER_HOST = 'docker.freeda.tech'
        NEXUS_DOCKER_URL = "https://${NEXUS_DOCKER_HOST}/repository/docker"
    }

    stages {
        stage('Build fluentd-kubernetes-daemonset for InsightOps') {
            steps {
                container('buildah') {
                    sh 'yum install make git -y'
                    echo "Building fluentd-kubernetes-daemonset for InsightOps version ${COMPLETE_VERSION}"
                    sh 'buildah login -u ${NEXUS_DOCKER_CREDS_USR} -p ${NEXUS_DOCKER_CREDS_PSW} ${NEXUS_DOCKER_URL}'
                    sh "make release-buildah no-cache=yes IMAGE_NAME=${NEXUS_DOCKER_HOST}/fluent/fluentd-kubernetes DOCKERFILE=${VERSION}/debian-insightops/Dockerfile VERSION=latest TAGS=${COMPLETE_VERSION}-debian-insightops-amd64-1.0,${VERSION}-debian-insightops-amd64-1,latest"
                }
            }
        }
    }

    post {
        success {
            container('buildah') {
                slackSend(color: '#54D843', message: "[SUCCESS] '${env.JOB_NAME}' -> commit '${SHORT_COMMIT}': pipeline ended successfully.")
            }
        }
        failure {
            container('buildah') {
                slackSend(color: '#F4322C', message: "[FAILED] '${env.JOB_NAME}' -> commit '${SHORT_COMMIT}': pipeline aborted.")
            }
        }
    }
}
