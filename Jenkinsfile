pipeline {
    agent any

    environment {
        DOCKERHUB_USER = "ali20052025"   //
        IMAGE_NAME     = "${DOCKERHUB_USER}/demo-image"
        IMAGE_TAG      = "${BUILD_NUMBER}"
        FULL_IMAGE     = "${IMAGE_NAME}:${IMAGE_TAG}"
    }

    stages {

        stage('Build Image') {
            steps {
                sh "docker build -f ${DOCKERFILE} -t ${FULL_IMAGE} ."
            }
        }

        stage('Scan with Trivy') {
            steps {
                sh """
                    trivy image \
                      --exit-code 1 \
                      --severity CRITICAL \
                      --no-progress \
                      ${FULL_IMAGE}
                """
            }
        }

        stage('Sign with Cosign') {
            steps {
                withCredentials([
                    string(credentialsId: 'cosign-private-key', variable: 'COSIGN_KEY'),
                    string(credentialsId: 'cosign-password',    variable: 'COSIGN_PASSWORD')
                ]) {
                    sh """
                        echo "\$COSIGN_KEY" > /tmp/cosign.key
                        COSIGN_PASSWORD=\$COSIGN_PASSWORD \
                        cosign sign --key /tmp/cosign.key \
                          --allow-insecure-registry \
                          ${FULL_IMAGE}
                        rm /tmp/cosign.key
                    """
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_PASS'
                )]) {
                    sh """
                        echo "\$DH_PASS" | docker login -u "\$DH_USER" --password-stdin
                        docker push ${FULL_IMAGE}
                    """
                }
            }
        }
    }

    post {
        failure {
            echo "Pipeline FAILED — image did not meet security policy. Not signed. Not pushed."
        }
        success {
            echo "Pipeline PASSED — image is clean, signed, and pushed to Docker Hub."
        }
    }
}
