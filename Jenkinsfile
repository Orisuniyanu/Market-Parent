pipeline {
    agent any
    stages {
        stage('Create context_env') {
            steps {
                sh '''
                    echo "GIT_URL=https://github.com/Orisuniyanu/Market-Parent.git" > context_env
                    echo "GIT_BRANCH=origin/main" >> context_env
                '''
            }
        }
        stage('Get Envs') {
            steps {
                script {
                    def lines = readFile("${env.WORKSPACE}/context_env").split("\n")
                    lines.each { envLine ->
                        def (key, value) = envLine.tokenize("=")
                        env."${key}" = "${value}"
                    }

                    env.PROJECT_NAME = env.GIT_URL.split("/")[4].replace(".git", "").trim()
                    env.PROJECT_BRANCH = env.GIT_BRANCH.split("/")[1].trim()

                    echo "PROJECT_NAME=${env.PROJECT_NAME}"
                    echo "PROJECT_BRANCH=${env.PROJECT_BRANCH}"
                }
            }
        }
        stage('Git Checkout') {
            steps {
                script {
                    echo "Checking out branch ${env.PROJECT_BRANCH} from ${env.GIT_URL}"
                    git url: env.GIT_URL,
                        credentialsId: 'github-credentials',
                        branch: env.PROJECT_BRANCH
                }
            }
        }
        stage('Docker Build & Push') {
            steps {
                echo "Building Docker image"
                sh "docker build -t orisuniyanu/${env.PROJECT_NAME}:${env.PROJECT_BRANCH}-${env.BUILD_NUMBER} ."

                withCredentials([usernamePassword(credentialsId: 'docker-hub-jenkins', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin 
                        docker push orisuniyanu/${env.PROJECT_NAME}:${env.PROJECT_BRANCH}-${env.BUILD_NUMBER}
                        echo "Docker image built successfully"
                        echo "Moving to the next stage"
                    """
                }
            }
        }
        stage('Deploy') {
            steps {
                withCredentials([
                    string(credentialsId: 'Remote_Host', variable: 'Remote_Host'),
                    string(credentialsId: 'Remote_User', variable: 'Remote_User')
                ]) {
                    echo "Deploying Docker image: orisuniyanu/${env.PROJECT_NAME}:${env.PROJECT_BRANCH}-${env.BUILD_NUMBER} to VM"

                    sshagent (credentials: ['ssh-vm-creds-id']) {
                        sh """
                            scp -o StrictHostKeyChecking=no docker-compose.yaml ${Remote_User}@${Remote_Host}:/home/${Remote_User}/docker-compose.yaml

                            ssh -o StrictHostKeyChecking=no ${Remote_User}@${Remote_Host} '
                                export PROJECT_NAME=${env.PROJECT_NAME}
                                export PROJECT_BRANCH=${env.PROJECT_BRANCH}
                                export BUILD_NUMBER=${env.BUILD_NUMBER}
                                
                                docker pull orisuniyanu/${env.PROJECT_NAME}:${env.PROJECT_BRANCH}-${env.BUILD_NUMBER} &&
                                docker compose -f /home/${Remote_User}/docker-compose.yaml down || true &&
                                docker compose -f /home/${Remote_User}/docker-compose.yaml up -d
                            '
                        """
                    }
                    echo "✅ Docker image deployed to your VM"
                }
            }
        }
    }
}
