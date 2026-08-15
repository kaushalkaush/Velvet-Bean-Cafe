pipeline {
    agent {
        label 'PRD-JNK02'
    }

    environment {
        DOCKER_IMAGE    = 'Velvet-Bean-Cafe:latest'
        CONTAINER_NAME  = 'Velvet-Bean-Cafe'
        HOST_PORT       = '8081'
        CONTAINER_PORT  = '8081'
    }

    stages {

        stage('Verify Source') {
            steps {
                echo "========================================"
                echo " Verifying GitHub Source"
                echo "========================================"

                sh '''
                    set -e

                    echo ""
                    echo "Hostname : $(hostname)"
                    echo "Workspace: $(pwd)"

                    echo ""
                    echo "Git branch:"
                    git branch --show-current || true

                    echo ""
                    echo "Git commit:"
                    git log -1 --oneline

                    echo ""
                    echo "Checking index.html..."

                    if [ ! -f "$WORKSPACE/index.html" ]; then
                        echo "ERROR: index.html not found in repository root."
                        exit 1
                    fi

                    echo "index.html found successfully."

                    echo ""
                    echo "File details:"
                    ls -lh "$WORKSPACE/index.html"

                    echo ""
                    echo "Checking Dockerfile..."

                    if [ -f "$WORKSPACE/Dockerfile" ]; then
                        echo "Dockerfile already exists."
                        ls -lh "$WORKSPACE/Dockerfile"
                    else
                        echo "Dockerfile will be created by the pipeline."
                    fi
                '''
            }
        }


        stage('Pre-Flight Checks') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'jenkinssvc',
                        usernameVariable: 'JENKINSVC_USER',
                        passwordVariable: 'JENKINSVC_PASSWORD'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "========================================"
                        echo " Jenkins Worker Pre-Flight Checks"
                        echo "========================================"

                        echo ""
                        echo "[1/6] Worker Information"

                        echo "Hostname       : $(hostname)"
                        echo "Jenkins User   : $(whoami)"
                        echo "Docker User    : $JENKINSVC_USER"
                        echo "Workspace      : $WORKSPACE"

                        echo ""
                        echo "[2/6] Checking Docker Installation"

                        if ! command -v docker >/dev/null 2>&1; then
                            echo "ERROR: Docker is not installed."
                            exit 1
                        fi

                        echo "Docker command found:"
                        docker --version

                        echo ""
                        echo "[3/6] Checking Docker Service"

                        if ! systemctl is-active --quiet docker; then
                            echo "ERROR: Docker service is NOT running."
                            exit 1
                        fi

                        echo "Docker service: RUNNING"

                        echo ""
                        echo "[4/6] Checking jenkinssvc Account"

                        if ! id "$JENKINSVC_USER" >/dev/null 2>&1; then
                            echo "ERROR: Account $JENKINSVC_USER does not exist."
                            exit 1
                        fi

                        echo "Account found:"
                        id "$JENKINSVC_USER"

                        echo ""
                        echo "[5/6] Checking Docker Access"

                        if ! echo "$JENKINSVC_PASSWORD" | \
                            su - "$JENKINSVC_USER" -c 'docker info' >/dev/null 2>&1; then

                            echo "ERROR: $JENKINSVC_USER cannot access Docker."
                            exit 1
                        fi

                        echo "Docker access: OK"

                        echo ""
                        echo "[6/6] Checking Workspace Access"

                        if ! echo "$JENKINSVC_PASSWORD" | \
                            su - "$JENKINSVC_USER" -c \
                            "test -r '$WORKSPACE/index.html'"; then

                            echo "ERROR: $JENKINSVC_USER cannot read index.html."
                            echo "Workspace: $WORKSPACE"
                            exit 1
                        fi

                        echo "Workspace access: OK"
                        echo "index.html readable by $JENKINSVC_USER"

                        echo ""
                        echo "========================================"
                        echo " PRE-FLIGHT CHECKS PASSED"
                        echo "========================================"
                    '''
                }
            }
        }


        stage('Create Dockerfile') {
            steps {

                sh '''
                    set -e

                    echo "========================================"
                    echo " Creating Dockerfile"
                    echo "========================================"

                    cat > "$WORKSPACE/Dockerfile" <<'EOF'
FROM python:3.12-alpine

WORKDIR /app

COPY index.html /app/index.html

EXPOSE 8000

CMD ["python", "-m", "http.server", "8000", "--bind", "0.0.0.0"]
EOF

                    echo ""
                    echo "Dockerfile created successfully."

                    echo ""
                    echo "Dockerfile contents:"
                    cat "$WORKSPACE/Dockerfile"
                '''
            }
        }


        stage('Build Docker Image') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'jenkinssvc',
                        usernameVariable: 'JENKINSVC_USER',
                        passwordVariable: 'JENKINSVC_PASSWORD'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "========================================"
                        echo " Building Docker Image"
                        echo "========================================"

                        echo ""
                        echo "Image     : $DOCKER_IMAGE"
                        echo "User      : $JENKINSVC_USER"
                        echo "Workspace : $WORKSPACE"

                        echo ""
                        echo "Verifying build context..."

                        echo "$JENKINSVC_PASSWORD" | \
                        su - "$JENKINSVC_USER" -c \
                        "test -r '$WORKSPACE/index.html'"

                        echo "index.html: READABLE"

                        echo "$JENKINSVC_PASSWORD" | \
                        su - "$JENKINSVC_USER" -c \
                        "test -r '$WORKSPACE/Dockerfile'"

                        echo "Dockerfile: READABLE"

                        echo ""
                        echo "Starting Docker build..."

                        echo "$JENKINSVC_PASSWORD" | \
                        su - "$JENKINSVC_USER" -c \
                        "docker build \
                        --tag '$DOCKER_IMAGE' \
                        '$WORKSPACE'"

                        echo ""
                        echo "========================================"
                        echo " Docker Image Built Successfully"
                        echo "========================================"

                        echo ""
                        echo "Image information:"

                        echo "$JENKINSVC_PASSWORD" | \
                        su - "$JENKINSVC_USER" -c \
                        "docker image inspect '$DOCKER_IMAGE' \
                        --format 'Repository: {{index .RepoTags 0}} | Size: {{.Size}} bytes'"

                        echo ""
                        echo "Docker images:"

                        echo "$JENKINSVC_PASSWORD" | \
                        su - "$JENKINSVC_USER" -c \
                        "docker images '$DOCKER_IMAGE'"
                    '''
                }
            }
        }


        stage('Deploy Container') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'jenkinssvc',
                        usernameVariable: 'JENKINSVC_USER',
                        passwordVariable: 'JENKINSVC_PASSWORD'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "========================================"
                        echo " Deploying Website Container"
                        echo "========================================"

                        echo ""
                        echo "Container : $CONTAINER_NAME"
                        echo "Image     : $DOCKER_IMAGE"
                        echo "Port      : $HOST_PORT:$CONTAINER_PORT"
                        echo "User      : $JENKINSVC_USER"

                        echo ""
                        echo "Checking if port $HOST_PORT is already in use..."

                        if ss -lnt | awk '{print $4}' | grep -q ":$HOST_PORT$"; then
                            echo "WARNING: Port $HOST_PORT is currently in use."
                            echo "Existing Docker container will be checked."
                        else
                            echo "Port $HOST_PORT is available."
                        fi

                        echo ""
                        echo "Removing existing container if present..."

                        echo "$JENKINSVC_PASSWORD" | \
                        su - "$JENKINSVC_USER" -c \
                        "docker rm -f '$CONTAINER_NAME' 2>/dev/null || true"

                        echo ""
                        echo "Starting new container..."

                        echo "$JENKINSVC_PASSWORD" | \
                        su - "$JENKINSVC_USER" -c \
                        "docker run -d \
                            --name '$CONTAINER_NAME' \
                            --restart unless-stopped \
                            --publish '$HOST_PORT:$CONTAINER_PORT' \
                            '$DOCKER_IMAGE'"

                        echo ""
                        echo "Container started successfully."

                        echo ""
                        echo "Container status:"

                        echo "$JENKINSVC_PASSWORD" | \
                        su - "$JENKINSVC_USER" -c \
                        "docker ps \
                            --filter name='$CONTAINER_NAME' \
                            --format 'table {{.Names}}\\t{{.Image}}\\t{{.Status}}\\t{{.Ports}}'"
                    '''
                }
            }
        }


        stage('Verify Container') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'jenkinssvc',
                        usernameVariable: 'JENKINSVC_USER',
                        passwordVariable: 'JENKINSVC_PASSWORD'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "========================================"
                        echo " Verifying Container"
                        echo "========================================"

                        sleep 3

                        echo ""
                        echo "Checking container status..."

                        CONTAINER_STATUS=$(echo "$JENKINSVC_PASSWORD" | \
                            su - "$JENKINSVC_USER" -c \
                            "docker inspect -f '{{.State.Status}}' '$CONTAINER_NAME'")

                        echo "Container status: $CONTAINER_STATUS"

                        if [ "$CONTAINER_STATUS" != "running" ]; then
                            echo "ERROR: Container is not running."

                            echo ""
                            echo "Container logs:"

                            echo "$JENKINSVC_PASSWORD" | \
                            su - "$JENKINSVC_USER" -c \
                            "docker logs '$CONTAINER_NAME'"

                            exit 1
                        fi

                        echo "Container: RUNNING"

                        echo ""
                        echo "Container details:"

                        echo "$JENKINSVC_PASSWORD" | \
                        su - "$JENKINSVC_USER" -c \
                        "docker ps \
                            --filter name='$CONTAINER_NAME' \
                            --format 'table {{.Names}}\\t{{.Image}}\\t{{.Status}}\\t{{.Ports}}'"
                    '''
                }
            }
        }


        stage('Verify Website') {
            steps {

                sh '''
                    set -e

                    echo "========================================"
                    echo " Verifying Website"
                    echo "========================================"

                    sleep 2

                    echo ""
                    echo "Testing HTTP endpoint..."

                    HTTP_CODE=$(curl -s \
                        -o /tmp/myrepo-index-test.html \
                        -w "%{http_code}" \
                        "http://127.0.0.1:$HOST_PORT/")

                    echo "HTTP Status: $HTTP_CODE"

                    if [ "$HTTP_CODE" != "200" ]; then
                        echo "ERROR: Website returned HTTP $HTTP_CODE"

                        echo ""
                        echo "Response:"
                        cat /tmp/myrepo-index-test.html || true

                        exit 1
                    fi

                    echo ""
                    echo "HTTP response: OK"

                    echo ""
                    echo "Checking index.html content..."

                    if ! grep -q "<html" /tmp/myrepo-index-test.html; then
                        echo "WARNING: HTTP response does not appear to contain standard HTML."
                    else
                        echo "HTML content detected."
                    fi

                    echo ""
                    echo "========================================"
                    echo " DEPLOYMENT SUCCESSFUL"
                    echo "========================================"

                    echo ""
                    echo "Website URL:"
                    echo "http://$(hostname -I | awk '{print $1}'):$HOST_PORT"

                    echo ""
                    echo "Local URL:"
                    echo "http://127.0.0.1:$HOST_PORT"
                '''
            }
        }
    }

    post {

        success {
            echo "========================================"
            echo " BUILD SUCCESSFUL"
            echo "========================================"
            echo "Website deployed successfully."
            echo "Container: Velvet-Bean-Cafe"
            echo "Port: 8081 -> 8081"
        }

        failure {
            echo "========================================"
            echo " BUILD FAILED"
            echo "========================================"
            echo "Check Jenkins Console Output."
        }

        always {
            echo "========================================"
            echo " Jenkins Build Completed"
            echo "========================================"
        }
    }
}
