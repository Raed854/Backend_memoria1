pipeline {
    agent any

    tools {
        jdk 'jdk-17'
        maven 'maven-3.9'
    }

    environment {
        IMAGE         = 'memoria-backend'
        REGISTRY      = 'docker.io'
        SONAR_ENV     = 'SonarQube'
        DOCKER_CREDS  = 'dockerhub'
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build & Test') {
            steps {
                sh './mvnw -B clean verify'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
                    archiveArtifacts artifacts: 'target/site/jacoco/**', allowEmptyArchive: true
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONAR_ENV}") {
                    sh './mvnw -B sonar:sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Archive JAR') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Docker Build & Push') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                    tag pattern: 'v\\d+\\.\\d+\\.\\d+', comparator: 'REGEXP'
                }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_CREDS}",
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_PASS'
                )]) {
                    sh '''
                        set -eu
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin "$REGISTRY"

                        TAG=$(git describe --tags --always)
                        SHA=$(git rev-parse --short HEAD)
                        IMG="$DH_USER/$IMAGE"

                        docker build -t "$IMG:$TAG" -t "$IMG:sha-$SHA" .

                        if [ "$BRANCH_NAME" = "main" ]; then
                            docker tag "$IMG:$TAG" "$IMG:latest"
                            docker push "$IMG:latest"
                        fi

                        docker push "$IMG:$TAG"
                        docker push "$IMG:sha-$SHA"

                        docker logout "$REGISTRY"
                    '''
                }
            }
        }
    }

    post {
        always { cleanWs() }
        success { echo "Build #${BUILD_NUMBER} OK — image publiée." }
        failure { echo "Build #${BUILD_NUMBER} en échec." }
    }
}
