pipeline {
  agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        spec:
          containers:
          - name: kaniko
            image: gcr.io/kaniko-project/executor:debug
            command: ["sleep"]
            args: ["9999999"]
            volumeMounts:
              - name: docker-config
                mountPath: /kaniko/.docker
          - name: kubectl
            image: bitnami/kubectl:latest
            command: ["sleep"]
            args: ["9999999"]
          volumes:
            - name: docker-config
              secret:
                secretName: dockerhub-cred
      '''
    }
  }

  triggers {
    githubPush()
  }

  environment {
    IMAGE_NAME = "medanyyy/pet-clinic"
    IMAGE_TAG  = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
  }

  stages {

    stage('Checkout') {
      steps {
        git url: 'https://github.com/Mahmoud-Medany/pet-clinic.git', branch: 'main'
      }
    }

    stage('Build & Push Image') {
      steps {
        container('kaniko') {
          sh """
            /kaniko/executor \
              --context=`pwd` \
              --destination=${IMAGE_NAME}:${IMAGE_TAG} \
              --destination=${IMAGE_NAME}:latest
          """
        }
      }
    }

    stage('Update Deploy Manifest') {
      steps {
        sh """
          sed -i "s|image: ${IMAGE_NAME}:.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|" deploy.yaml
        """
      }
    }

    stage('Commit Updated Manifest') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'github-cred', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
          sh """
            git config user.email "mahmmoudmedany213@gmail.com"
            git config user.name "Mahmoud-Medany"
            git add deploy.yaml
            git commit -m "ci: bump image to ${IMAGE_TAG}"
            git push https://${GIT_USER}:${GIT_PASS}@github.com/Mahmoud-Medany/pet-clinic.git HEAD:main
          """
        }
      }
    }

    stage('Deploy to Cluster') {
      steps {
        container('kubectl') {
          sh "kubectl set image deployment/petclinic petclinic=${IMAGE_NAME}:${IMAGE_TAG} -n petclinic"
        }
      }
    }
  }
}