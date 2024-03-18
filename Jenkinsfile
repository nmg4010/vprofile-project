def COLOR_MAP = [
    SUCCESS: '#36a64f',
    FAILURE: '#ff0000',
    ABORTED: '#cccccc',
    UNSTABLE: '#ffcc00',
]

pipeline {
    agent any

    environment {
        SNAP_REPO = 'vprofile-snapshot'
        NEXUS_USER = credentials('nexus-credentials-id').username
        NEXUS_PASS = credentials('nexus-credentials-id').password
        RELEASE_REPO = 'vprofile-release'
        CENTRAL_REPO = 'vpro-maven-central'
        NEXUS_IP = '34.125.126.192'
        NEXUS_PORT = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group'
        SONARSERVER = 'sonarserver'
        SONARSCANNER = 'sonarscanner'
    }

    stages {
        stage('Build'){
            steps {
                script {
                    withMaven(
                        maven: 'MAVEN3',
                        jdk: 'OracleJDK11'
                    ) {
                        sh 'mvn -s settings.xml -DskipTests install'
                    }
                }
            }
            post {
                success {
                    echo 'Archiving'
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    withMaven(
                        maven: 'MAVEN3',
                        jdk: 'OracleJDK11'
                    ) {
                        sh 'mvn -s settings.xml test'
                    }
                }
            }
        }

        stage('Checkstyle Analysis') {
            steps {
                script {
                    withMaven(
                        maven: 'MAVEN3',
                        jdk: 'OracleJDK11'
                    ) {
                        sh 'mvn -s settings.xml checkstyle:checkstyle'
                    }
                }
            }
        }

        stage ('Sonar Analysis') {
            steps {
                script {
                    withSonarQubeEnv(SONARSERVER) {
                        sh "${tool(SONARSCANNER)}/bin/sonar-scanner \
                            -Dsonar.projectKey=vprofile \
                            -Dsonar.projectName=vprofile \
                            -Dsonar.projectVersion=1.0 \
                            -Dsonar.sources=src/ \
                            -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                            -Dsonar.junit.reportsPath=target/surefire-reports/ \
                            -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                            -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml"
                    }
                }
            }
        }

        stage ("Quality Gate") {
            steps {
                script {
                    timeout(time: 1, unit: 'HOURS') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        stage ("Upload Artifact") {
            steps {
                script {
                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: "${NEXUS_IP}:${NEXUS_PORT}",  
                        groupId: 'QA',
                        version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                        repository: RELEASE_REPO,
                        credentialsId: "${NEXUS_LOGIN}", 
                        artifacts: [
                            [artifactId: 'vproapp',
                            classifier: '',
                            file: 'target/vprofile-v2.war',
                            type: 'war']
                        ]
                    )
                }
            }
        }
    }
    post {
        always {
            echo 'Slack Notifications'
            slackSend channel: '#cicd',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }
}
