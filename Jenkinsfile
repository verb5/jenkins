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
                container('buildah') {
                    unstash 'builtApp'
                    sh '''
                        echo "Building Docker image..."
                        ls  -al
                        ls -al build
                        cat /etc/os-release
                        pwd
                    '''
                }
            }
        }
    }
}
