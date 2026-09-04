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
                        echo 'Skipping generated manifest-update commit.'
                    }
                }
            }
        }

        stage('Verify GitHub Credentials') {
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
                        set +x
                        set -eu

                        command -v python3 >/dev/null

                        python3 - <<'PY'
import json
import os
import sys
import urllib.error
import urllib.request

username = os.environ["GIT_USERNAME"].strip()
token = os.environ["GIT_TOKEN"]

if username != "princedevops08":
    sys.exit("ERROR: Jenkins credential username must be princedevops08.")

if not token or token != token.strip():
    sys.exit("ERROR: Token is empty or contains surrounding whitespace.")

request = urllib.request.Request(
    "https://api.github.com/user",
    headers={
        "Authorization": "Bearer " + token,
        "Accept": "application/vnd.github+json",
        "User-Agent": "Jenkins-credential-check"
    }
)

try:
    with urllib.request.urlopen(request, timeout=30) as response:
        account = json.load(response)["login"]
except urllib.error.HTTPError as error:
    sys.exit(
        "ERROR: GitHub credential check returned HTTP "
        + str(error.code)
        + ". Check token validity and access restrictions."
    )
except urllib.error.URLError:
    sys.exit("ERROR: Cannot connect to GitHub API. Check network/DNS.")

print("Authenticated GitHub account:", account)

if account != "princedevops08":
    sys.exit("ERROR: The token belongs to the wrong GitHub account.")

print("GitHub token is valid. Repository write access is checked during push.")
PY
                    '''
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
                        set +x
                        set -eu

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
                        set +x
                        set -eu

                        git config user.name "$GIT_USER"
                        git config user.email "$GIT_EMAIL"

                        python3 - <<'PY'
import os
import pathlib
import re
import sys

path = pathlib.Path("k8s/deployment.yml")
content = path.read_text()
image = os.environ["IMAGE_NAME"]
tag = os.environ["IMAGE_TAG"]

pattern = (
    r"(?m)^(\\s*image:\\s*)"
    + re.escape(image)
    + r"(?::[^\\s#]+)?\\s*$"
)

updated, count = re.subn(
    pattern,
    lambda match: match.group(1) + image + ":" + tag,
    content
)

if count != 1:
    sys.exit("ERROR: Expected exactly one matching application image.")

path.write_text(updated)
PY

                        git add k8s/deployment.yml

                        if git diff --cached --quiet; then
                            echo "No manifest changes to commit."
                            exit 0
                        fi

                        git commit \
                            -m "Updated image to ${IMAGE_TAG} [skip ci]"

                        export GIT_TERMINAL_PROMPT=0

                        git -c credential.helper= \
                            -c 'credential.helper=!f() { printf "%s\\n" "username=$GIT_USERNAME" "password=$GIT_TOKEN"; }; f' \
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
            echo 'Pipeline failed. Check the first failed stage.'
        }
    }
}
