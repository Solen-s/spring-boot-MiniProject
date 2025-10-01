pipeline {
    agent { label 'jenkins-jenkins-agent' }
    tools { 
        maven 'maven'
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
                    checkout scm
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                container('kaniko') {
                    withCredentials([usernamePassword(
                        credentialsId: 'b30738c2-998e-4b66-aaf8-462eb6e651a6', 
                        usernameVariable: 'DOCKERHUB_USERNAME', 
                        passwordVariable: 'DOCKERHUB_PASSWORD'
                    )]) {
                        sh '''#!/bin/sh
mkdir -p /workspace/.docker
cat > /workspace/.docker/config.json <<EOF
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "$(echo -n $DOCKERHUB_USERNAME:$DOCKERHUB_PASSWORD | base64)"
    }
  }
}
EOF

/kaniko/executor \
  --dockerfile /workspace/Dockerfile \
  --context /workspace \
  --destination $REGISTRY:$IMAGE_TAG \
  --skip-tls-verify=true
'''
                    }
                }
            }
        }

        stage('Update Helm Values') {
            steps {
                container('git') {
                    withCredentials([usernamePassword(
                        credentialsId: '41a9fcbf-6233-428f-9eff-c1e8f4b27790', 
                        usernameVariable: 'GIT_USER', 
                        passwordVariable: 'GIT_TOKEN'
                    )]) {
                        script {
                            try {
                                sh '''#!/bin/sh
git config --global user.email "solen0918@gmail.com"
git config --global user.name "Solen-s"
rm -rf helm-spring-boot-repo || true
git clone https://$GIT_USER:$GIT_TOKEN@github.com/Solen-s/Manifest-Spring-boot.git helm-spring-boot-repo
cd helm-spring-boot-repo
sed -i "s|tag:.*|tag: $IMAGE_TAG|" values.yaml
git add values.yaml
git commit -m "Update image tag to $IMAGE_TAG" || echo "No changes to commit"
git push origin main
'''
                                echo "✅ Helm values updated successfully."
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
            echo "💥 Pipeline failed. Check logs above."
        }
    }
}
