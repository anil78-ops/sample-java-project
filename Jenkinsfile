pipeline {
  agent any

  parameters {
    string(name: 'BRANCH_NAME', defaultValue: 'dev', description: 'Git branch to build and deploy')
  }

  environment {
    IMAGE_NAME = "kubernetes-demo"
    DOCKER_REGISTRY = "krishnasravi"
    GIT_CREDENTIALS_ID = "github-pat"
    APP_DIR = "app"
  }

  stages {
    stage('Clone Repository') {
      steps {
        git branch: "${params.BRANCH_NAME}",
            url: 'https://github.com/krishnasravi/kubernetes-demo.git',
            credentialsId: "${GIT_CREDENTIALS_ID}"
      }
    }

    stage('Maven Build') {
      steps {
        dir("${APP_DIR}") {
          sh 'mvn clean package -DskipTests'
        }
      }
    }

    stage('Docker Build and Push') {
      steps {
        dir("${APP_DIR}") {
          script {
            def safeTag = params.BRANCH_NAME.replaceAll('/', '-')
            def imageTag = "${safeTag}-${BUILD_NUMBER}"
            env.IMAGE_TAG = imageTag

            withDockerRegistry(credentialsId: 'docker-cred-id', url: '') {
              sh """
                docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} .
                docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
              """
            }
          }
        }
      }
    }

    stage('Kubernetes Deploy') {
      steps {
        script {
          def namespace = params.BRANCH_NAME.toLowerCase()
          def deploymentFile = ""
          def kubeconfigCredentialId = ""
          def safeTag = params.BRANCH_NAME.replaceAll('/', '-')
          def imageTag = "${safeTag}-${BUILD_NUMBER}"

          switch (params.BRANCH_NAME) {
            case 'dev':
              kubeconfigCredentialId = 'kubeconfig-dev'
              deploymentFile = 'manifests/dev/dev-deployment.yaml'
              break
            case 'uat':
              kubeconfigCredentialId = 'kubeconfig-uat'
              deploymentFile = 'manifests/uat/uat-deployment.yaml'
              break
            case 'main':
              kubeconfigCredentialId = 'kubeconfig-prod'
              deploymentFile = 'manifests/prod/prod-deployment.yaml'
              break
            default:
              error "Unsupported branch for deployment: ${params.BRANCH_NAME}"
          }

          withCredentials([file(credentialsId: kubeconfigCredentialId, variable: 'KCFG')]) {
            sh """
              export KUBECONFIG=${KCFG}
              sed -i 's|image: .*|image: ${DOCKER_REGISTRY}/${IMAGE_NAME}:${imageTag}|' ${deploymentFile}
              kubectl apply --insecure-skip-tls-verify -f ${deploymentFile}
              kubectl rollout status --insecure-skip-tls-verify deployment/k8s-demo -n ${namespace}
            """
          }
        }
      }
    }
  }
}
