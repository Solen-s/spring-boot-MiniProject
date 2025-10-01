pipeline {
    agent {     
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: docker
      image: docker:24.0.5
      command:
        - cat
      tty: true
      volumeMounts:
        - name: docker-sock
          mountPath: /var/run/docker.sock
        - name: workspace-volume
          mountPath: /workspace

    - name: git
      image: alpine/git
      command:
        - cat
      tty: true
      workingDir: /workspace
      volumeMounts:
        - name: workspace-volume
          mountPath: /workspace

  volumes:
    - name: workspace-volume
      emptyDir: {}
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
"""
        }
    }

    environment {
        REGISTRY = "solenn9/spring-boot"
        IMAGE_TAG = "${BUILD_NUMBER}"
        HELM_REPO = "https://github.com/Solen-s/Manifest-Spring-boot.git"
        HELM_VALUES_FILE = "values.yaml"
    }

    stages {
        stage('Checkout App') {
            steps {
                container('git') {
                    echo "🔹 Checking out source code..."
                    checkout scm
                    echo "✅ Checkout complete."
                    sh 'ls -la /workspace'  // debug: see if Dockerfile exists
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                container('docker') {
                        withCredentials([usernamePassword(
                            credentialsId: 'docker-token', 
                            usernameVariable: 'DOCKERHUB_USERNAME', 
                            passwordVariable: 'DOCKERHUB_PASSWORD'
                        )]) {
                            script {
                                sh '''
                                 pwd
                                '''
                                echo "🔹 Building and pushing Docker image..."
                                try {
                                    sh '''
                                        echo "$DOCKERHUB_PASSWORD" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin
                                        docker build -t ${REGISTRY}:${IMAGE_TAG} .
                                        docker push ${REGISTRY}:${IMAGE_TAG}
                                        docker logout
                                    '''
                                    echo "✅ Docker image built and pushed successfully: ${REGISTRY}:${IMAGE_TAG}"
                                } catch (err) {
                                    echo "❌ Docker build/push failed!"
                                    error("Stopping pipeline due to Docker error.")
                                }
                            }
                            
                        }
                    
                   
                }
            }
        }

        stage('Update Helm Values') {
            steps {
                container('git') {
                    withCredentials([usernamePassword(
                            credentialsId: 'git_token', 
                            usernameVariable: 'GIT_USER', 
                            passwordVariable: 'GIT_TOKEN'
                        )]) {
                        script{
                            echo "🔹 Updating Helm values..."
                            try {
                                sh '''
                                     git config --global user.email "solen0918@gmail.com"
                                git config --global user.name "Solen-s"
                                rm -rf helm-spring-boot-repo || true
                                git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/Solen-s/Manifest-Spring-boot.git helm-spring-boot-repo
                                cd helm-spring-boot-repo
                                sed -i 's|tag:.*|tag: "'$IMAGE_TAG'"|' values.yaml
                                git add values.yaml
                                git commit -m "Update image tag to '$IMAGE_TAG'" || echo "No changes to commit"
                                git push origin main
                                '''
                                echo "✅ Helm values updated and pushed successfully."
                            } catch (err) {
                                echo "❌ Updating Helm values failed!"
                                error("Stopping pipeline due to Git/Helm error.")
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "🎉 Pipeline completed successfully!"
        }
        failure {
            echo "💥 Pipeline failed! Check stage logs above."
        }
    }
}
