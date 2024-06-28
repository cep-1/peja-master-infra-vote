pipeline {
    agent any
    environment {
        GITHUB_TOKEN = credentials('peja-master-infra-github-lm')
        AWS_CREDENTIALS_ID = 'peja-master-infra-user'
        AWS_REGION = 'eu-central-1'
        ECR_URL = '211125460791.dkr.ecr.eu-central-1.amazonaws.com/peja-vote-repo'
    }
    tools {
        dockerTool 'docker'
    }
    triggers {
        githubPush()
    }
    stages {
        stage('Determine Build Type') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        BUILD_TYPE = 'push'
                    }
                    else {
                        BUILD_TYPE = 'test'
                    }
                }
            }
        }
        stage('Connect to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: "${AWS_CREDENTIALS_ID}",
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh 'aws ecr get-login-password --region ${AWS_REGION} | sudo docker login --username AWS --password-stdin ${ECR_URL}'
                }
            }
        }
        stage('Docker Config and Build Image') {
            steps {
                sh 'sudo docker build -t peja-master-infra-vote .'
            }
        }
        stage('Handle Push Event') {
            steps {
                script {
                    if (BUILD_TYPE == 'push') {
                        def releaseTag = env.release_tag_name
                        sh "sudo docker tag peja-master-infra-vote:${releaseTag} ${ECR_URL}:${releaseTag}"
                        sh "sudo docker tag peja-master-infra-vote:latest ${ECR_URL}:latest"
                        sh "sudo docker push ${ECR_URL}:${releaseTag}"
                        sh "sudo docker push ${ECR_URL}:latest"
                    }   
                }
            }
        }
    }
}
