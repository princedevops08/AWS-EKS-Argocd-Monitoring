pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    environment {
        IMAGE_NAME = 'princedevops08/multibranch-flask-app'
        GIT_USER   = 'princedevops08'
        GIT_EMAIL  = 'princedevops08@gmail.com'
        GIT_REPO   = 'https://github.com/princedevops08/AWS-EKS-Argocd-Monitoring.git'
        SKIP_CI    = 'false'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm

                script {
                    env.IMAGE_TAG = "build-${BUILD_NUMBER}"

                    def commitMessage = sh(
                        script: 'git log -1 --pretty=%B',
                        returnStdout: true
                    ).trim()

                    if (commitMessage.contains('[skip ci]')) {
                        env.SKIP_CI = 'true'
                        currentBuild.description = 'Skipped: [skip ci]'
                        echo 'Skipping image build and manifest update.'
                    }
                }
            }
        }

        stage('Build Image') {
            when {
                allOf {
                    branch 'main'
                    expression { env.SKIP_CI != 'true' }
                }
            }

            steps {
                sh '''
                    set -eu
                    docker build -t "$IMAGE_NAME:$IMAGE_TAG" .
                '''
            }
        }

        stage('Push Image') {
            when {
                allOf {
                    branch 'main'
                    expression { env.SKIP_CI != 'true' }
                }
            }

            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-cred',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x

                        # Use temporary credentials for this shell only.
                        DOCKER_CONFIG="$(mktemp -d)"
                        export DOCKER_CONFIG

                        cleanup() {
                            docker logout >/dev/null 2>&1 || true
                            rm -f "$DOCKER_CONFIG/config.json"
                            rmdir "$DOCKER_CONFIG" 2>/dev/null || true
                        }
                        trap cleanup EXIT

                        printf '%s' "$DOCKER_PASS" |
                            docker login \
                                --username "$DOCKER_USER" \
                                --password-stdin

                        docker push "$IMAGE_NAME:$IMAGE_TAG"
                    '''
                }
            }
        }

        stage('Update K8s Manifest') {
            when {
                allOf {
                    branch 'main'
                    expression { env.SKIP_CI != 'true' }
                }
            }

            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x

                        git config user.name "$GIT_USER"
                        git config user.email "$GIT_EMAIL"

                        test -f k8s/deployment.yml

                        sed -i \
                            "s|image:.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|" \
                            k8s/deployment.yml

                        git add k8s/deployment.yml

                        if git diff --cached --quiet; then
                            echo "No manifest changes to commit."
                            exit 0
                        fi

                        git commit \
                            -m "Updated image to ${IMAGE_TAG} [skip ci]"

                        export GIT_TERMINAL_PROMPT=0

                        git -c credential.helper= \
                            -c 'credential.helper=!f() { printf "%s\\\\n" "username=$GIT_USERNAME" "password=$GIT_TOKEN"; }; f' \
                            push "$GIT_REPO" HEAD:main
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check the failed stage in Console Output.'
        }
    }
}
