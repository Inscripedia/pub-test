pipeline {
    agent {
        kubernetes {
            label 'jenkins-slave'
        }
    }
    environment {
        GIT_SSH_KEY_ID_APP = credentials('github.test-cicd.deploy')
        GIT_SSH_KEY_ID_K8S = credentials('github.test-k8s.deploy')
    }
    stages {
        
        stage('Prepare SSH') {
            steps {
                script {
                    // Fetch GitHub's SSH key and add it to known_hosts
                    sh '''
                    mkdir -p ~/.ssh
                    ssh-keyscan github.com >> ~/.ssh/known_hosts
                    '''
                }
            }
        }
        
        stage('docker login') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'DOCKER_CREDS') {
                        sh(script: "docker info", returnStdout: true).trim().split('\n').each { line ->
                            echo line
                        }
                    }
                }
            }
        }

        stage('git clone app') {
            steps {
                script {
                    sshagent(credentials: ['github.test-cicd.deploy']) {
                        sh(script: """
                            git clone git@github.com:Inscripedia/test-cicd.git
                        """, returnStdout: true)
                    }
                }
            }
        }
        
        stage('git clone k8s') {
            steps {
                script {
                    sshagent(credentials: ['github.test-k8s.deploy']) {
                        sh(script: """
                            git clone git@github.com:Inscripedia/test-k8s.git
                        """, returnStdout: true)
                    }
                }
            }
        }

        stage('docker build') {
            steps {
                sh script: '''
                #!/bin/bash
                cd $WORKSPACE/test-cicd/python
                docker build . --network host -t synic04/jenkins:${BUILD_NUMBER}
                '''
            }
        }

        stage('docker push') {
            steps {
                script {
                    // Authenticate and push the Docker image
                    docker.withRegistry('https://index.docker.io/v1/', 'DOCKER_CREDS') {
                        sh(script: """
                            docker push synic04/jenkins:${BUILD_NUMBER}
                        """)
                    }
                }
            }
        }

        stage('deploy') {
            steps {
                sh script: '''
                #!/bin/bash
                cd $WORKSPACE/test-k8s/
                # Get kubectl for this demo
                curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
                chmod +x ./kubectl
        
                
                ./kubectl set image deployment/example-deploy example-app=synic04/jenkins:${BUILD_NUMBER}
                ./kubectl apply -f ./services/service.yaml
                ./kubectl get deployment example-deploy -o yaml
                '''
            }
        }


    }
}
