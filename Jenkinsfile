pipeline {
    agent any

    tools {
        jdk 'jdk-21'
        nodejs 'node-js'
    }

    environment {
        SCANNER_HOME = tool 'sonar'
        DOCKER_IMAGE = 'sunmarsh/bms:latest'
        EKS_CLUSTER_NAME = 'sunmarshcluster'
        AWS_REGION = 'us-west-1'
    }
    
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main', credentialsId: 'github-creds', url: 'https://github.com/Marshsun/Book-My-Show.git'
                sh 'ls -la'  // Verify files after checkout
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' 
                    $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=Book-My-Show-DevOps-Project \
                        -Dsonar.projectName='Book-My-Show DevOps Project' 
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    // Added timeout to avoid infinite waiting
                    timeout(time: 1, unit: 'MINUTES') {
                        def qg = waitForQualityGate abortPipeline: true, credentialsId: 'sonar-creds'
                        echo "Quality Gate status: ${qg.status}"
                    }
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                cd bookmyshow-app
                ls -la  # Verify package.json exists
                if [ -f package.json ]; then
                    rm -rf node_modules package-lock.json  # Remove old dependencies
                    npm install  # Install fresh dependencies
                else
                    echo "Error: package.json not found in bookmyshow-app!"
                    exit 1
                fi
                '''
            }
        }

        stage('OWASP FS Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy FS Scan') {
    steps {
        sh '''
        if ! command -v trivy &> /dev/null
        then
            echo "Trivy not installed! Skipping scan."
            exit 0
        fi

        echo "Running Trivy FS scan..."
        trivy fs . > trivyfs.txt
        '''
    }
}


        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub-creds', toolName: 'docker') {
                        sh ''' 
                        echo "Building Docker image..."
                        docker build --no-cache -t $DOCKER_IMAGE -f bookmyshow-app/Dockerfile bookmyshow-app

                        echo "Pushing Docker image to Docker Hub..."
                        docker push $DOCKER_IMAGE
                        '''
                    }
                }
            }
        }

        stage('Deploy to EKS Cluster') {
            steps {
                script {
                    sh '''
                    echo "Verifying AWS credentials..."
                    aws sts get-caller-identity

                    echo "Configuring kubectl for EKS cluster..."
                    aws eks update-kubeconfig --name $EKS_CLUSTER_NAME --region $AWS_REGION

                    echo "Verifying kubeconfig..."
                    kubectl config view

                    echo "Deploying application to EKS..."
                    kubectl apply -f deployment.yml
                    kubectl apply -f service.yml

                    echo "Verifying deployment..."
                    kubectl get pods
                    kubectl get svc
                    '''
                }
            }
        }
    }

    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>" +
                      "Build Number: ${env.BUILD_NUMBER}<br/>" +
                      "URL: ${env.BUILD_URL}<br/>",
                to: 'sunmarshkumar2002@gmail.com',
                attachmentsPattern: 'trivyfs.txt'
        }
    }
}