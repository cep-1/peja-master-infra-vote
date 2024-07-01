pipeline {
    agent any
    environment {
        GITHUB_TOKEN = credentials('peja-master-infra-github-lm')
    }
    tools {
        dockerTool 'docker'
    }
    triggers {
        githubPush()
    }
    stages {
        stage('Determine Event Type') {
            steps {
                script {
                    def isReleaseEvent = false
                    try {
                        def response = sh(script: """
                            curl -s -H "Authorization: token ${GITHUB_TOKEN}" \
                            "https://api.github.com/repos/cep-1/peja-master-infra-vote/releases/latest" \
                            """, returnStdout: true).trim()
                        def json = new groovy.json.JsonSlurper().parseText(response)
                        if (json?.tag_name) {
                            isReleaseEvent = true
                            env.RELEASE_TAG = json.tag_name
                        }
                    } catch (Exception e) {
                        echo "Error fetching release information: ${e}"
                    }
                    env.IS_RELEASE_EVENT = isReleaseEvent.toString()
                }
            }
        }
        stage('Connect to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: "peja-master-infra-user",
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh 'aws ecr get-login-password --region eu-central-1 | sudo docker login --username AWS --password-stdin 211125460791.dkr.ecr.eu-central-1.amazonaws.com'
                }
            }
        }
        stage('Docker Config and Build Image') {
            steps {
                sh 'sudo docker build -t peja-master-infra-vote .'
            }
        }
        stage('Handle Release Event') {
            when {
                expression { env.IS_RELEASE_EVENT == 'true' }
            }
            steps {
                script {
                    sh '''
                        #!/bin/bash
                        echo ${GITHUB_TOKEN}
                        RELEASE_TAG=$(curl -s -H "Authorization: token ${GITHUB_TOKEN}" \
                        "https://api.github.com/repos/cep-1/peja-master-infra-vote/releases/latest" \
                        | jq -r .tag_name')

                        echo "Latest Release Tag: $RELEASE_TAG"
                        echo "RELEASE_TAG=$RELEASE_TAG" > release_tag.txt
                    '''
                    env.RELEASE_TAG = readFile('release_tag.txt').trim()
                }
                sh 'sudo docker tag peja-master-infra-vote:${RELEASE_TAG} 211125460791.dkr.ecr.eu-central-1.amazonaws.com/peja-vote-repo:${RELEASE_TAG}'
                sh 'sudo docker tag peja-master-infra-vote:latest 211125460791.dkr.ecr.eu-central-1.amazonaws.com/peja-vote-repo:latest'
                sh 'sudo docker push 211125460791.dkr.ecr.eu-central-1.amazonaws.com/peja-vote-repo:${RELEASE_TAG}'
                sh 'sudo docker push 211125460791.dkr.ecr.eu-central-1.amazonaws.com/peja-vote-repo:latest'
            }
        }
    }
}
