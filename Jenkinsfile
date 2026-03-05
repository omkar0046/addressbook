pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'SonarScanner'

        DOCKER_REGISTRY = "98.81.97.26:8081"
        DOCKER_REPO     = "docker-hosted"

        IMAGE_NAME = "myapp"
        IMAGE_TAG  = "${env.BUILD_NUMBER}-${env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : 'latest'}"

        DOCKER_IMAGE        = "${DOCKER_REGISTRY}/${DOCKER_REPO}/${IMAGE_NAME}:${IMAGE_TAG}"
        DOCKER_IMAGE_LATEST = "${DOCKER_REGISTRY}/${DOCKER_REPO}/${IMAGE_NAME}:latest"

        SONAR_PROJECT_KEY = "my-app"
        SONAR_HOST_URL    = "http://98.81.97.26:9000"

        GIT_REPO_URL = "https://github.com/omkar0046/addressbook.git"
        GIT_BRANCH   = "master"
    }

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['dev', 'qa', 'prod'], description: 'Select deployment environment')
        booleanParam(name: 'SKIP_SONAR', defaultValue: false, description: 'Skip SonarQube analysis')
    }

    stages {

        stage('Cleanup Workspace') {
            steps { cleanWs() }
        }

        stage('Git Checkout') {
            steps {
                git branch: "${GIT_BRANCH}",
                    credentialsId: 'git-credentials',
                    url: "${GIT_REPO_URL}"
            }
        }

        stage('SonarQube Analysis') {
            when { expression { !params.SKIP_SONAR } }
            steps {
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.projectName=myapp \
                        -Dsonar.sources=k8s \
                        -Dsonar.inclusions=**/*.yaml \
                        -Dsonar.host.url=${SONAR_HOST_URL} \
                        -Dsonar.token=${SONAR_TOKEN}
                    """
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                    docker build -t ${DOCKER_IMAGE} \
                                 -t ${DOCKER_IMAGE_LATEST} .
                """
            }
        }

        stage('Docker Push to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-credentials',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh """
                        echo "${NEXUS_PASS}" | docker login ${DOCKER_REGISTRY} -u "${NEXUS_USER}" --password-stdin
                        docker push ${DOCKER_IMAGE}
                        docker push ${DOCKER_IMAGE_LATEST}
                        docker logout ${DOCKER_REGISTRY}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh """
                        kubectl --kubeconfig=${KUBECONFIG_FILE} set image deployment/myapp myapp=${DOCKER_IMAGE} -n ${params.DEPLOY_ENV} || true
                        kubectl --kubeconfig=${KUBECONFIG_FILE} apply -f k8s/deployment.yaml -n ${params.DEPLOY_ENV}
                        kubectl --kubeconfig=${KUBECONFIG_FILE} rollout status deployment/myapp -n ${params.DEPLOY_ENV} --timeout=120s
                    """
                }
            }
        }
    }

    post {
        always {
            sh """
                docker rmi ${DOCKER_IMAGE} 2>/dev/null || true
                docker rmi ${DOCKER_IMAGE_LATEST} 2>/dev/null || true
                docker image prune -f || true
            """
            cleanWs()
        }

        success {
            echo "✅ PIPELINE SUCCESS"
        }

        failure {
            echo "❌ PIPELINE FAILED"
        }
    }
}
