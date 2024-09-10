pipeline {
    agent any
    environment {
        AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        DOCKER_IMAGE_TAG      = "suguslove10/finance-me-microservice:v1"
    }
    stages {
        stage('Clone Git Repository') {
            steps {
                git 'https://github.com/suguslove10/finance-me-microservice.git'
            }
        }
        stage('Build Maven Project') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${DOCKER_IMAGE_TAG} --cache-from=${DOCKER_IMAGE_TAG} ."
                    sh 'docker images'
                }
            }
        }
        stage('Push Docker Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'Docker-cred', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                    sh "echo $PASS | docker login -u $USER --password-stdin"
                    sh "docker push ${DOCKER_IMAGE_TAG}"
                }
            }
        }
        stage('Terraform Init') {
            steps {
                script {
                    sh '''
                    terraform workspace select test || terraform workspace new test
                    terraform init -input=false
                    '''
                }
            }
        }
        stage('Apply Terraform Only If Needed') {
            steps {
                script {
                    def changes = sh(returnStdout: true, script: "terraform plan -detailed-exitcode").trim()
                    if (changes == '2') {
                        echo 'Changes detected. Applying Terraform changes...'
                        sh "terraform apply -auto-approve tfplan"
                    } else {
                        echo 'No changes detected in the infrastructure.'
                    }
                }
            }
        }
        stage('Run Ansible Playbook') {
            steps {
                script {
                    def instanceIp = readFile('instance_ip.txt').trim()
                    sh """
                        ansible-playbook -i inventory.ini -e "prometheus_ip=${instanceIp}" ansible-playbook.yml
                    """
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline executed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
