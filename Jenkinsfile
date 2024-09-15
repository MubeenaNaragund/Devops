pipeline {
    agent any

    environment {
        REPO_URL = 'https://github.com/your-username/your-repo.git'
        BRANCH_NAME = 'main'
        AWS_INSTANCE_IP = 'your-ec2-public-ip'
        AWS_REGION = 'us-east-1'
        KUBECONFIG_PATH = '/path/to/your/kubeconfig' // Path to the kubeconfig file
        DOCKER_REGISTRY = 'your-docker-repo' // Docker registry URL
        IMAGE_NAME = 'my-java-app'
        SSH_KEY_PATH = '/path/to/your/private-key.pem' // Path to your SSH private key
        SSH_USER = 'ec2-user'  // Or your EC2 user
    }

    stages {
        stage('Checkout') {
            steps {
                // Pull code from Git repository
                git branch: "${BRANCH_NAME}", url: "${REPO_URL}"
            }
        }

        stage('Build and Test') {
            steps {
                script {
                    // Ensure Maven is installed on the Jenkins agent
                    def mvnHome = tool name: 'Maven 3', type: 'maven'

                    // Build the application using Maven
                    withEnv(["M2_HOME=${mvnHome}"]) {
                        // Build the application and skip tests in the build phase
                        sh 'mvn clean package -DskipTests'
                        
                        // Run tests
                        def testResult = sh script: 'mvn test', returnStatus: true
                        if (testResult != 0) {
                            error "Tests failed. Aborting deployment."
                        }
                    }

                    // Archive test results and build artifacts
                    junit '**/target/test-*.xml'
                    archiveArtifacts artifacts: '**/target/*.jar', allowEmptyArchive: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Build Docker image
                    sh """
                    docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest .
                    """

                    // Push Docker image to Docker registry
                    sh """
                    docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    // Copy Kubernetes configuration file to Jenkins agent
                    sh """
                    scp -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ${KUBECONFIG_PATH} ${SSH_USER}@${AWS_INSTANCE_IP}:/home/${SSH_USER}/kubeconfig
                    """
                    
                    // Deploy to Kubernetes
                    sh """
                    ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ${SSH_USER}@${AWS_INSTANCE_IP} << 'EOF'
                    export KUBECONFIG=/home/${SSH_USER}/kubeconfig
                    kubectl apply -f /home/${SSH_USER}/deployment.yaml
                    kubectl apply -f /home/${SSH_USER}/service.yaml
                    EOF
                    """
                }
            }
        }
    }

    post {
        always {
            // Clean up workspace
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
