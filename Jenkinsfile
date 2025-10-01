pipeline {
    agent {     
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:debug
      command:
        - /busybox/sh
      args:
        - -c
        - sleep 999999
      tty: true
      workingDir: /workspace
      volumeMounts:
        - name: workspace-volume
          mountPath: /workspace

    - name: git
      image: alpine/git
      command:
        - /bin/sh
      args:
        - -c
        - sleep 999999
      tty: true
      workingDir: /workspace
      volumeMounts:
        - name: workspace-volume
          mountPath: /workspace

  volumes:
    - name: workspace-volume
      emptyDir: {}
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
                container('kaniko') {
                    echo "🔹 Starting Kaniko build..."
                    catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                        withCredentials([usernamePassword(
                            credentialsId: 'docker-token', 
                            usernameVariable: 'DOCKERHUB_USERNAME', 
                            passwordVariable: 'DOCKERHUB_PASSWORD'
                        )]) {
                            sh '''#!/bin/sh
                            pwd
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
                    echo "✅ Kaniko build finished."
                }
            }
        }

        stage('Update Helm Values') {
            steps {
                container('git') {
                    echo "🔹 Updating Helm values..."
                    catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                        withCredentials([usernamePassword(
                            credentialsId: 'git_token', 
                            usernameVariable: 'GIT_USER', 
                            passwordVariable: 'GIT_TOKEN'
                        )]) {
                            sh '''#!/bin/sh
                               pwd
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
                        }
                    }
                    echo "✅ Helm values updated successfully."
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
