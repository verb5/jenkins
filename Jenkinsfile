def buildahPodYaml = '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: buildah
    image: quay.io/buildah/stable:latest
    command: ['/bin/bash', '-c', 'sleep infinity']
'''

def commonPodYaml = '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: node
    image: node:25-alpine
    command: ['sleep']
    args: ['infinity']
  securityContext:
    runAsUser: 0
'''

pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: buildah
    image: quay.io/buildah/stable:latest
    command: ['/bin/bash', '-c', 'sleep infinity']
    securityContext:
      runAsUser: 0
  - name: shell
    image: ubuntu
    command:
    - sleep
    args:
    - infinity
    securityContext:
      # ubuntu runs as root by default, it is recommended or even mandatory in some environments (such as pod security admission "restricted") to run as a non-root user.
      runAsUser: 1000
'''
            // Can also wrap individual steps:
            // container('shell') {
            //     sh 'hostname'
            // }
            defaultContainer 'shell'
            retries 2
        }
    }
    stages {
        stage('Main') {
                steps {
                    sh '''
                    echo "Running inside a Kubernetes Pod"
                    cat /etc/os-release
                '''
                }
        }
        stage('Using Node container') {
            agent {
                kubernetes {
                    yaml commonPodYaml
                    defaultContainer 'node'
                }
            }
            steps {
                // script {
                //     if (fileExists('src/main/resources/index.html')) {
                //         echo 'File src/main/resources/index.html found!'
                //     }
                // }
                sh '''
                    echo "Running inside the Debian container"
                    cat /etc/os-release
                    node --version
                    npm --version
                    npm ci
                    npm run build
                '''
                stash includes: 'build/**', name: 'builtApp'
            }
        }
        stage('Build docker image') {
            steps {
                // 1. Retrieve artifacts first
                unstash 'builtApp'

                // 2. Access credentials and execute commands
                container('buildah') {
                    withCredentials([
                usernamePassword(
                    credentialsId: '8c6a5efa-fc5f-4c5f-a6c8-0a0147c33bef',
                    usernameVariable: 'DOCKERHUB_USERNAME',
                    passwordVariable: 'DOCKERHUB_PASSWORD'
                )
            ]) {
                        sh '''
                    echo "Building Docker image..."

                    # ----------------------------------------------------
                    # CRITICAL FIX: Set the VFS storage driver environment variable.
                    # This ensures Buildah knows to run in PSS-compliant rootless mode
                    # before it attempts the restricted CLONE_NEWUSER operation.
                    export BUILDAH_STORAGE_DRIVER=vfs

                    # 1. Build the image (VFS is now set via ENV var, so we can remove the flag from the command)
                    buildah bud --format=docker -t verb5/frontend:$BUILD_NUMBER -f container/Dockerfile .

                    # 2. Log in
                    buildah login -u $DOCKERHUB_USERNAME -p $DOCKERHUB_PASSWORD docker.io

                    # 3. Push the image
                    buildah push verb5/frontend:$BUILD_NUMBER docker://verb5/frontend:$BUILD_NUMBER

                    echo "Image push complete."
                '''
            }
                }
            }
        }
    }
}
