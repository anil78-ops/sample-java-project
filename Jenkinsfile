pipeline {
  agent any

  environment {
    IMAGE_NAME = "kubernetes-demo" // Replace with your app name
    DOCKER_REGISTRY = "krishnasravi" // Replace with your DockerHub user/org
    GIT_CREDENTIALS_ID = "github-pat" // Jenkins Git credentials ID
  }

  stages {
    stage('Clone Repository') {
      steps {
        git branch: "${env.BRANCH_NAME}", url: 'https://github.com/krishnasravi/kubernetes-demo.git', credentialsId: "${GIT_CREDENTIALS_ID}"
      }
    }

    stage('Maven Build') {
      steps {
        sh 'mvn clean package -DskipTests'
      }
    }

    stage('Docker Build and Push') {
      steps {
        script {
          def tag = BRANCH_NAME.replaceAll("/", "-")
          env.IMAGE_TAG = "${tag}-${BUILD_NUMBER}"
          withDockerRegistry([credentialsId: 'docker-cred-id', url: 'https://index.docker.io/v1/'])
          sh """
            docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} .
            docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
          """
        }
      }
    }

    stage('Kubernetes Deploy') {
      steps {
        script {
          def kubeconfig = ""
          def deploymentFile = ""
          def namespace = BRANCH_NAME.toLowerCase() 

          if (env.BRANCH_NAME == 'dev') {
            kubeconfig = credentials('kubeconfig-dev')
            deploymentFile = 'k8s/dev/dev-deployment.yaml'
          } else if (env.BRANCH_NAME == 'uat') {
            kubeconfig = credentials('kubeconfig-uat')
            deploymentFile = 'k8s/uat/uat-deployment.yaml'
          } else if (env.BRANCH_NAME == 'main') {
            kubeconfig = credentials('kubeconfig-prod')
            deploymentFile = 'k8s/prod/prod-deployment.yaml'
          } else {
            error "Unsupported branch for deployment: ${BRANCH_NAME}"
          }

          // Update deployment file with new image tag
          sh """
            sed -i 's|image: .*|image: ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}|' ${deploymentFile}
            export KUBECONFIG=${kubeconfig}
            kubectl apply -f ${deploymentFile}
            kubectl rollout status deployment/${IMAGE_NAME} -n ${namespace}
          """
        }
      }
    }
  }
}
