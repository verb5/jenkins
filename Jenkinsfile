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
        stage('Using Node container'){
            agent {
                kubernetes {
                    yaml commonPodYaml
                    defaultContainer 'node'
                }
            }
            steps{
                sh '''
                    echo "Running inside the Debian container"
                    cat /etc/os-release
                    node --version
                    npm --version
                '''
            }
        }
    }
}
