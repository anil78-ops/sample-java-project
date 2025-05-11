pipeline {
  agent any

  parameters {
    string(name: 'BRANCH_NAME', defaultValue: 'dev', description: 'Git branch to build and deploy')
  }

  environment {
    IMAGE_NAME = "kubernetes-demo"              // Application name
    DOCKER_REGISTRY = "krishnasravi"            // DockerHub username or registry
    GIT_CREDENTIALS_ID = "github-pat"           // GitHub credentials ID
    APP_DIR = "app"                             // Subdirectory where app code resides
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

          // Inject kubeconfig content as a file
          withCredentials([file(credentialsId: kubeconfigCredentialId, variable: 'KUBECONFIG_FILE')]) {
            sh """
              sed -i 's|image: .*|image: ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}|' ${deploymentFile}
              export KUBECONFIG=$KUBECONFIG_FILE
              kubectl apply -f ${deploymentFile}
              kubectl rollout status deployment/${IMAGE_NAME} -n ${namespace}
            """
          }
        }
      }
    }
  }
}
