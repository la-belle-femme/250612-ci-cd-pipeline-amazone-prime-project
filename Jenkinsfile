pipeline {
    agent any
    
    environment {
        // DockerHub credentials
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKER_IMAGE = 'bettysami/amazon-prime-clone'
        SONARQUBE_SERVER = 'SonarQube'
        SONARQUBE_TOKEN = credentials('sonarqube-token')
        
        // Build information
        BUILD_TIMESTAMP = sh(script: "date '+%Y%m%d-%H%M%S'", returnStdout: true).trim()
    }
    
    tools {
        nodejs 'NodeJS-18'
    }
    
    stages {
        stage('🔄 1. Git Checkout') {
            steps {
                echo '🔄 Checking out source code from GitHub...'
                checkout scm
                
                script {
                    // Get git information
                    env.GIT_COMMIT_SHORT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    env.GIT_BRANCH_NAME = sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()
                }
                
                sh '''
                    echo "=== Git Information ==="
                    git --version
                    echo "Branch: $GIT_BRANCH_NAME"
                    echo "Commit: $GIT_COMMIT_SHORT"
                    echo "Repository contents:"
                    ls -la
                '''
            }
        }
        
        stage('📦 2. Install Dependencies') {
            steps {
                echo '📦 Installing Node.js dependencies...'
                sh '''
                    echo "=== Node.js Information ==="
                    node --version
                    npm --version
                    
                    echo "=== Installing Dependencies ==="
                    npm install
                    
                    echo "=== Dependency Installation Complete ==="
                    ls -la node_modules/ | head -10
                '''
            }
        }
        
        stage('🔍 3. SonarQube Analysis') {
            environment {
                SCANNER_HOME = tool 'SonarQubeScanner'
            }
            steps {
                echo '🔍 Running SonarQube code quality analysis...'
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        echo "=== SonarQube Analysis ==="
                        echo "Scanner Home: $SCANNER_HOME"
                        echo "SonarQube Server URL: $SONAR_HOST_URL"
                        
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=amazon-prime-clone \
                        -Dsonar.projectName="Amazon Prime Clone" \
                        -Dsonar.projectVersion=$BUILD_NUMBER \
                        -Dsonar.sources=src \
                        -Dsonar.sourceEncoding=UTF-8 \
                        -Dsonar.exclusions="node_modules/**,public/**,k8s_files/**,terraform_code/**,*.log" \
                        -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                        -Dsonar.typescript.lcov.reportPaths=coverage/lcov.info
                    '''
                }
            }
        }
        
        stage('🎯 4. Quality Gate') {
            steps {
                echo '🎯 Waiting for SonarQube Quality Gate...'
                timeout(time: 10, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        } else {
                            echo "✅ Quality Gate passed with status: ${qg.status}"
                        }
                    }
                }
            }
        }
        
        stage('🔒 5. Security Scan - File System') {
            steps {
                echo '🔒 Performing filesystem security scan with Trivy...'
                sh '''
                    echo "=== Trivy File System Scan ==="
                    trivy --version
                    
                    # Scan filesystem for vulnerabilities
                    trivy fs . \
                        --format table \
                        --output trivy-fs-report.txt \
                        --severity HIGH,CRITICAL \
                        --exit-code 0
                    
                    echo "=== Security Scan Results ==="
                    cat trivy-fs-report.txt
                    
                    # Count vulnerabilities
                    HIGH_COUNT=$(grep -c "HIGH" trivy-fs-report.txt || echo "0")
                    CRITICAL_COUNT=$(grep -c "CRITICAL" trivy-fs-report.txt || echo "0")
                    
                    echo "Found $HIGH_COUNT HIGH and $CRITICAL_COUNT CRITICAL vulnerabilities"
                '''
                
                // Archive the security report
                archiveArtifacts artifacts: 'trivy-fs-report.txt', allowEmptyArchive: true, fingerprint: true
            }
        }
        
        stage('🐳 6. Docker Build & Tag') {
            steps {
                echo '🐳 Building Docker image...'
                script {
                    def imageTag = "${BUILD_NUMBER}"
                    def timestampTag = "${BUILD_TIMESTAMP}"
                    def commitTag = "commit-${GIT_COMMIT_SHORT}"
                    
                    sh """
                        echo "=== Docker Build Information ==="
                        docker --version
                        docker info | grep -i "Storage Driver"
                        
                        echo "=== Building Docker Images ==="
                        # Build with multiple tags
                        docker build -t ${DOCKER_IMAGE}:${imageTag} .
                        docker build -t ${DOCKER_IMAGE}:latest .
                        docker build -t ${DOCKER_IMAGE}:${timestampTag} .
                        docker build -t ${DOCKER_IMAGE}:${commitTag} .
                        
                        echo "=== Docker Images Built ==="
                        docker images | grep ${DOCKER_IMAGE}
                        
                        # Get image size
                        IMAGE_SIZE=\$(docker images ${DOCKER_IMAGE}:${imageTag} --format "table {{.Size}}" | tail -1)
                        echo "Image size: \$IMAGE_SIZE"
                    """
                    
                    // Store tags for later use
                    env.IMAGE_TAG = imageTag
                    env.TIMESTAMP_TAG = timestampTag
                    env.COMMIT_TAG = commitTag
                }
            }
        }
        
        stage('🔍 7. Docker Security Scan') {
            steps {
                echo '🔍 Scanning Docker image for vulnerabilities...'
                sh '''
                    echo "=== Docker Image Security Scan ==="
                    
                    # Scan the built Docker image
                    trivy image \
                        --format table \
                        --output trivy-image-report.txt \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        ${DOCKER_IMAGE}:${BUILD_NUMBER}
                    
                    echo "=== Docker Image Scan Results ==="
                    cat trivy-image-report.txt
                    
                    # Count vulnerabilities in image
                    IMAGE_HIGH_COUNT=$(grep -c "HIGH" trivy-image-report.txt || echo "0")
                    IMAGE_CRITICAL_COUNT=$(grep -c "CRITICAL" trivy-image-report.txt || echo "0")
                    
                    echo "Docker image has $IMAGE_HIGH_COUNT HIGH and $IMAGE_CRITICAL_COUNT CRITICAL vulnerabilities"
                '''
                
                // Archive the security report
                archiveArtifacts artifacts: 'trivy-image-report.txt', allowEmptyArchive: true, fingerprint: true
            }
        }
        
        stage('📤 8. Push to DockerHub') {
            steps {
                echo '📤 Pushing Docker image to DockerHub...'
                script {
                    sh '''
                        echo "=== DockerHub Login ==="
                        echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
                        
                        echo "=== Pushing Images to DockerHub ==="
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        docker push ${DOCKER_IMAGE}:latest
                        docker push ${DOCKER_IMAGE}:${TIMESTAMP_TAG}
                        docker push ${DOCKER_IMAGE}:${COMMIT_TAG}
                        
                        echo "=== Successfully Pushed All Tags ==="
                        echo "✅ ${DOCKER_IMAGE}:${BUILD_NUMBER}"
                        echo "✅ ${DOCKER_IMAGE}:latest"
                        echo "✅ ${DOCKER_IMAGE}:${TIMESTAMP_TAG}"
                        echo "✅ ${DOCKER_IMAGE}:${COMMIT_TAG}"
                        
                        # Logout for security
                        docker logout
                    '''
                }
            }
        }
        
        stage('📝 9. Update Kubernetes Manifests') {
            steps {
                echo '📝 Updating Kubernetes deployment manifests for GitOps...'
                script {
                    sh '''
                        echo "=== Updating K8s Manifests ==="
                        
                        if [ -d "k8s_files" ]; then
                            echo "Found k8s_files directory"
                            ls -la k8s_files/
                            
                            # Update deployment.yaml if it exists
                            if [ -f "k8s_files/deployment.yaml" ]; then
                                echo "Updating deployment.yaml with new image tag"
                                
                                # Backup original
                                cp k8s_files/deployment.yaml k8s_files/deployment.yaml.bak
                                
                                # Update image tag
                                sed -i "s|image: .*amazon-prime-clone.*|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|g" k8s_files/deployment.yaml
                                
                                echo "Updated deployment.yaml:"
                                grep -A 2 -B 2 "image:" k8s_files/deployment.yaml || echo "No image line found"
                            else
                                echo "deployment.yaml not found in k8s_files/"
                            fi
                            
                            # List all files that might need updating
                            find k8s_files/ -name "*.yaml" -o -name "*.yml" | head -5
                            
                        else
                            echo "⚠️  k8s_files directory not found"
                            echo "This will be created in Phase 2 (CD with ArgoCD)"
                        fi
                    '''
                }
            }
        }
    }
    
    post {
        always {
            echo '🧹 Cleaning up workspace and Docker images...'
            sh '''
                echo "=== Cleanup Process ==="
                
                # Clean up local Docker images to free space
                docker image prune -f --filter "dangling=true"
                
                # Remove old images (keep last 3 builds)
                OLD_IMAGES=$(docker images ${DOCKER_IMAGE} --format "{{.Repository}}:{{.Tag}}" | grep -v latest | tail -n +4)
                if [ ! -z "$OLD_IMAGES" ]; then
                    echo "Removing old images: $OLD_IMAGES"
                    echo "$OLD_IMAGES" | xargs docker rmi -f || true
                fi
                
                # Show disk usage
                echo "=== Docker Disk Usage ==="
                docker system df
                
                echo "=== Cleanup Complete ==="
            '''
            
            // Archive important artifacts
            archiveArtifacts artifacts: '*.txt, k8s_files/*.yaml', allowEmptyArchive: true, fingerprint: true
            
            // Clean workspace
            cleanWs()
        }
        
        success {
            echo '✅ CI Pipeline completed successfully!'
            script {
                def buildSummary = """
                🎉 BUILD SUCCESS! 🎉
                
                📋 Build Details:
                ├─ Build Number: #${BUILD_NUMBER}
                ├─ Repository: ${env.GIT_URL}
                ├─ Branch: ${env.GIT_BRANCH_NAME}
                ├─ Commit: ${env.GIT_COMMIT_SHORT}
                ├─ Build Time: ${BUILD_TIMESTAMP}
                └─ Duration: ${currentBuild.durationString}
                
                🐳 Docker Images Published:
                ├─ ${DOCKER_IMAGE}:${BUILD_NUMBER}
                ├─ ${DOCKER_IMAGE}:latest  
                ├─ ${DOCKER_IMAGE}:${env.TIMESTAMP_TAG}
                └─ ${DOCKER_IMAGE}:${env.COMMIT_TAG}
                
                🔗 Links:
                ├─ Jenkins Build: ${BUILD_URL}
                ├─ Console Output: ${BUILD_URL}console
                ├─ SonarQube: http://52.91.102.229:9000/dashboard?id=amazon-prime-clone
                └─ DockerHub: https://hub.docker.com/r/la-belle-femme/amazon-prime-clone
                
                🚀 Ready for Phase 2: EKS Deployment with ArgoCD!
                """
                
                echo buildSummary
                
                // You can add Slack/Email notifications here later
            }
        }
        
        failure {
            echo '❌ CI Pipeline failed!'
            script {
                def failureSummary = """
                💥 BUILD FAILED! 💥
                
                📋 Failure Details:
                ├─ Build Number: #${BUILD_NUMBER}
                ├─ Repository: ${env.GIT_URL}
                ├─ Branch: ${env.GIT_BRANCH_NAME}
                ├─ Failed Stage: ${env.STAGE_NAME}
                └─ Build URL: ${BUILD_URL}
                
                🔍 Troubleshooting:
                ├─ Check Console Output: ${BUILD_URL}console
                ├─ Review Security Scans: Check archived reports
                ├─ Verify Dependencies: Check npm install logs
                └─ SonarQube Issues: http://52.91.102.229:9000
                
                💡 Next Steps:
                ├─ Fix the reported issues
                ├─ Commit your changes
                └─ Trigger a new build
                """
                
                echo failureSummary
            }
        }
        
        unstable {
            echo '⚠️ Build completed with warnings'
            echo 'Check the console output and quality reports for details'
        }
    }
}
