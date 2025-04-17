pipeline {
    agent any
    tools {
        jdk 'java17'
        nodejs 'node16'
    }
    environment {
        AWS_ACCESS_KEY_ID     = credentials('aws-access-credential-key')  
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-credential-key')
        AWS_REGION            = 'us-east-1' 
        AWS_ECR_REPO          = '286668306280.dkr.ecr.us-east-1.amazonaws.com/musthafa'
        IMAGE_NAME            = 'game-2048'
        IMAGE_TAG             = "latest"
        EKS_CLUSTER           = 'dev-k8'
        EKS_NAMESPACE         = 'default'
        GIT_REPO_URL          = 'https://github.com/musthi1995/sample-app-2048.git'
        SCANNER_HOME          = tool 'sonar-scanner'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout') {
            steps {
                script {
                    echo "Checking out the repository..."
                    git url: "${env.GIT_REPO_URL}", branch: 'master', credentialsId: 'git-credential'
                }
            }
        }

        stage('Trigger Info') {
            steps {
                wrap([$class: 'BuildUser']) {
                    script {
                        echo "Build triggered by: ${BUILD_USER}"
                    }
                }
            }
        }

        
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=DevSecOps \
                        -Dsonar.projectKey=DevSecOps \
                        -Dsonar.host.url=http://54.162.161.22:9000/
                    '''
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: true, qualityGate: 'devsecops', credentialsId: 'sonar-token-01'
                }
            }
        }

        stage("Generate SonarQube Audit Report") {
            steps {
                withCredentials([string(credentialsId: 'sonar-token-01', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        curl -u $SONAR_TOKEN: \
                        "http://54.162.161.22:9000/api/measures/component?component=DevSecOps&metricKeys=bugs,vulnerabilities,code_smells,coverage,sqale_index,security_hotspots" \
                        -o sonar-audit-report.json
                    '''
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }

        stage('Docker Image Build') {
            steps {
                script {
                    sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG .'
                }
            }
        }

        stage('TRIVY Scan') {
            steps {
                script {
                    sh 'trivy image $IMAGE_NAME:$IMAGE_TAG > trivy.json'
               }
           }
        }

        stage('Push to AWS ECR') {
            steps {
                script {
                    sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 286668306280.dkr.ecr.us-east-1.amazonaws.com'
                    sh 'docker tag $IMAGE_NAME:$IMAGE_TAG $AWS_ECR_REPO:$IMAGE_TAG'
                    sh 'docker push $AWS_ECR_REPO:$IMAGE_TAG'
                }
            }
        }

        stage('Update Kubeconfig') {
            steps {
                sh '''
                    aws eks update-kubeconfig \
                    --region $AWS_REGION \
                    --name $EKS_CLUSTER
                '''
            }
        }

        stage('Deploy to EKS with Helm') {
            steps {
                dir('react-app-helm') {
                    sh '''
                        helm upgrade --install react-app . \
                        --namespace $EKS_NAMESPACE \
                        --set image.repository=$AWS_ECR_REPO \
                        --set image.tag=$IMAGE_TAG
                    '''
                }
            }
        }
        stage('Wait for Trivy Operator Scan)') {
            steps {
                echo 'Waiting 1 minutes for Trivy Operator to scan workloads...'
                sh 'sleep 60'
            }
        }

        stage('Fetch Trivy Operator Reports') {
            steps {
                sh '''
                    echo "Fetching Trivy Operator vulnerability and config audit reports..."
                    kubectl get vulnerabilityreports -A -o json > trivy-operator-vulns.json
                    kubectl get configauditreports -A -o json > trivy-operator-misconfigs.json
                '''
            }
        }
        
    }

    post {
        success {
            echo 'Deployment successful'
        }
        failure {
            echo 'Deployment failed'
        }
        always {
            archiveArtifacts artifacts: 'trivy.json, sonar-audit-report.json, trivy-operator-*.json', onlyIfSuccessful: false
        }
    }
}
