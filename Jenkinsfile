pipeline {
    agent any
    tools {
        jdk 'jdk8'
        nodejs 'node9'
        
    }
    environment {
        //be sure to replace "bhavukm" with your own Docker Hub username
        DOCKER_IMAGE_NAME = "dheerajparikh/train-schedule"
    }
    stages {
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build(DOCKER_IMAGE_NAME)
                    app.inside {
                        sh 'echo Hello, World!'
                    }
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('CanaryDeploy') {
            when {
                branch 'master'
            }
            environment { 
                CANARY_REPLICAS = 1
            }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                      kubectl config set-context $(kubectl config current-context)

                      envsubst < train-schedule-kube-canary.yml | tee train-schedule-kube-canary.yml
                      kubectl apply -f train-schedule-kube-canary.yml
                    '''
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            environment { 
                CANARY_REPLICAS = 0
            }
            steps {
                input 'Deploy to Production?'
                milestone(1)
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                      kubectl config set-context $(kubectl config current-context)

                      envsubst < train-schedule-kube-canary.yml | tee train-schedule-kube-canary.yml
                      kubectl apply -f train-schedule-kube-canary.yml

                      envsubst < train-schedule-kube.yml | tee train-schedule-kube.yml
                      kubectl apply -f train-schedule-kube.yml
                    '''
                }
            }
        }
    }
}
