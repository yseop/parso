#! /usr/bin/env groovy

@Library('Utilities') _

pipeline {
    agent {
        node {
            label 'java'
        }
    }

    options {
        disableConcurrentBuilds()
        timeout time: 30, unit: 'MINUTES'
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '10'))
    }

    environment {
        NEXUS                       = credentials("nexus3")
        NEXUS_USER                  = "${NEXUS_USR}"
        NEXUS_PASSWORD              = "${NEXUS_PSW}"

        DOCKER_BUILD_TAG            = utils.getDockerBuildTag()
        DOCKER_TAG                  = utils.getDockerTag()

        SLACK_NOTIFY                = utils.getAuthorSlackHandle()
    }

    stages {
        stage('Abort previous build') {
            steps {
                script {
                    utils.killPreviousBuilds()
                }
            }
        }

        stage('Set Maven version') {
            steps {
                script {
                    // Set Maven version dynamically depending on what is built
                    def newVersion
                    if (env.TAG_NAME) {
                        // Building a tag
                        newVersion = DOCKER_TAG
                    } else {
                        newVersion = DOCKER_BUILD_TAG
                    }
                    sh([
                        './mvnw -s .settings.xml',
                        "-B versions:set -DnewVersion='${newVersion}'",
                        '-DprocessAllModules',
                        '-DskipTests',
                    ].join(' '))
                }
            }
        }

        stage('Build') {
            steps {
                sh './mvnw -s .settings.xml -B package'
            }
        }

        stage('Deploy artifacts') {
            when {
                buildingTag()
            }
            steps {
                // Publish artifacts on Yseop Nexus, by-passing the declared OSS Nexus repository.
                // Using deploy:deploy is mandatory to not use what is declared in the deploy goal (or the nexus staging plugin disables the maven deploy plugin)
                // Using jar:jar is necessary so that deploy:deploy gets enough context to know what to deploy.
                sh './mvnw -B -s .settings.xml -DaltDeploymentRepository=yseop_thirdparty::default::https://nexus3.yseop-hosting.com/repository/yseop_thirdparty jar:jar deploy:deploy '
            }
        }
    }

    post {
        always {
            // xunit reports
            xunit(
                thresholdMode: 1, // 1 for number, 2 for percent
                testTimeMargin: '3000',
                thresholds: [skipped(failureThreshold: '0'), failed(failureThreshold: '0')],
                tools: [JUnit(pattern: 'target/surefire-reports/*.xml', skipNoTestFiles: true)]
            )
        }
        success {
            // Archive the JAR application.
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true

            slackSend color: 'good', message: "SUCCESSFUL: Job “<${env.BUILD_URL}|${env.JOB_NAME} [${env.BUILD_NUMBER}]>” <@${SLACK_NOTIFY}>"
        }
        failure {
            slackSend color: 'danger', message: "FAILED: Job “<${env.BUILD_URL}|${env.JOB_NAME} [${env.BUILD_NUMBER}]>” <@${SLACK_NOTIFY}>"
        }
        cleanup {
            script {
                // Clean up our workspace.
                cleanWs deleteDirs: true
                deleteDir()

                utils.cleanupJenkinsAgent()
            }
        }
    }
}
