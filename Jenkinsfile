def podTemplate = """
apiVersion: v1
kind: Pod
metadata:
labels:
  jenkins/jenkins-jenkins-agent: "true"
  jenkins/label: "jenkins-jenkins-agent"
spec:
  securityContext:
    fsGroup: 1950    # Group ID of docker group on k8s nodes.
  serviceAccountName: jenkins
  containers:
  - name: topgun
    image: human537/inbound-agent:v1
    command:
    - cat
    tty: true
    volumeMounts:
    - mountPath: "/home/jenkins/agent"
      name: "workspace-volume"
      readOnly: false
    - mountPath: /var/run/docker.sock
      name: docker-sock
      readOnly: true
    workingDir: "/home/jenkins/agent"
  - name: golang
    image: golang:1.10
    command:
    - cat
    tty: true
    workingDir: "/home/jenkins/agent"
    volumeMounts:
    - mountPath: "/home/jenkins/agent"
      name: "workspace-volume"
      readOnly: false
  nodeSelector:
    kubernetes.io/os: "linux"
  restartPolicy: "Never"
  volumes:
  - name: "workspace-volume"
    emptyDir:
      medium: ""    
  - name: docker-sock
    hostPath:
      path: "/var/run/docker.sock"
"""

pipeline {
    agent {
        kubernetes {
/*          label 'sample-app' */
          yaml podTemplate
          defaultContainer 'topgun'
        }
    }
    environment {
        DOCKER_IMAGE_NAME = "hmmun99/frontend"
    }
    stages {
        stage('checkout'){
            steps{
                git branch: 'main', url: 'https://github.com/hmmun99/microservices-demo.git'
            }
       
        }
        stage('Build Docker Image') {
            when {
                branch 'main'
            }
            steps {
                container('topgun') {
                    sh ''
                    script {
                        app = docker.build(DOCKER_IMAGE_NAME,"--network host ./src/frontend/")
                        app.inside {
                            sh 'echo Hello, World!123'
                        }
                    }
                    echo 'Running Build Docker Image'
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'main'
            }
            steps {     
                container('topgun') {
                    script {
                        docker.withRegistry('https://registry.hub.docker.com', 'docker') {
                            app.push("${env.BUILD_NUMBER}")
                            app.push("latest")
                        }
                    }
                    echo 'Running Push Docker Image'
                }
            }
        }     
        stage('Canary') {
            when {
                branch 'canary'
            }
           steps {
//                withKubeConfig([credentialsId: 'kubeconfig']) {
                    container('topgun') {
                        sh 'curl -LO "https://dl.k8s.io/release/v1.24.0/bin/linux/amd64/kubectl"'
                        sh 'chmod u+x ./kubectl'
                        sh """
                           ./kubectl patch deployment frontend -n canary -p \
                           '{"spec":{"template":{"spec":{"containers":[{"name":"server","image":"${DOCKER_IMAGE_NAME}:latest"}]}}}}'
                           """
//               milestone(1)
              }          
           }
        }
        stage('DeployToProduction') {
            when {
                branch 'main'
            }
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    container('topgun') {
                        sh 'curl -LO "https://dl.k8s.io/release/v1.24.0/bin/linux/amd64/kubectl"'
                        sh 'chmod u+x ./kubectl'
                        sh """
                           ./kubectl patch deployment frontend -n product -p \
                           '{"spec":{"template":{"spec":{"containers":[{"name":"server","image":"${DOCKER_IMAGE_NAME}:latest"}]}}}}'
                           """
                       }
                   }
               }
           }
       }
   }
