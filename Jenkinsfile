pipeline {
    agent any

    tools {
        jdk 'JDK17'
        maven 'Maven3'
    }

    environment {
        SONAR_SCANNER = tool 'sonar-scanner'
        IMAGE_NAME = "your-dockerhub-username/java-springboot"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Code Checkout') {
            steps {
                git credentialsId: 'github-token', url: 'https://github.com/your-username/java-springboot.git', branch: 'main'
            }
        }

        stage('Maven Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=java-springboot \
                        -Dsonar.projectName="Java Spring Boot App"
                    '''
                }
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.txt .'
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.txt ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                        docker push ${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        kubectl set image deployment/java-app java-app=${IMAGE_NAME}:${IMAGE_TAG}
                        kubectl rollout status deployment/java-app
                    '''
                }
            }
        }
    }

    post {
    always {
        emailext(
            subject: "Pipeline ${currentBuild.result}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: "Build ${currentBuild.result}. Check: ${env.BUILD_URL}",
            to: 'your-email@gmail.com'
        )
    }
}
}
