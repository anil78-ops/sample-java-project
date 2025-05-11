pipeline {
  agent any

  environment {
    IMAGE_NAME = "kubernetes-demo"
    DOCKER_REGISTRY = "krishnasravi"
    GIT_CREDENTIALS_ID = "github-pat"
    APP_DIR = "app"
  }

  stages {
    stage('Determine Branch') {
      steps {
        script {
          env.ACTUAL_BRANCH = env.BRANCH_NAME ?: env.GIT_BRANCH?.replaceFirst(/^origin\//, '') ?: 'dev'
          echo "🔀 Detected branch: ${env.ACTUAL_BRANCH}"
        }
      }
    }

    stage('Clone Repository') {
      steps {
        git branch: "${env.ACTUAL_BRANCH}",
            url: 'https://github.com/krishnasravi/kubernetes-demo.git',
            credentialsId: "${GIT_CREDENTIALS_ID}"
      }
    }

    stage('Maven Build') {
      when {
        expression { return ['dev', 'uat', 'main'].contains(env.ACTUAL_BRANCH) }
      }
      steps {
        dir("${APP_DIR}") {
          sh 'mvn clean package -DskipTests'
        }
      }
    }

    stage('Docker Build and Push') {
      when {
        expression { return ['dev', 'uat', 'main'].contains(env.ACTUAL_BRANCH) }
      }
      steps {
        dir("${APP_DIR}") {
          script {
            def safeTag = env.ACTUAL_BRANCH.replaceAll('/', '-')
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
      when {
        expression { return ['dev', 'uat', 'main'].contains(env.ACTUAL_BRANCH) }
      }
      steps {
        script {
          def branch = env.ACTUAL_BRANCH
          def namespace = ['dev': 'dev', 'uat': 'uat', 'main': 'prod'][branch]
          def deploymentFile = ""
          def kubeconfigCredentialId = ""
          def safeTag = branch.replaceAll('/', '-')
          def imageTag = "${safeTag}-${BUILD_NUMBER}"

          switch (branch) {
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
              error "Unsupported branch for deployment: ${branch}"
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

  post {
    always {
      script {
        def safeTag = env.ACTUAL_BRANCH.replaceAll('/', '-')
        def imageTag = "${safeTag}-${BUILD_NUMBER}"

        echo "🧹 Running cleanup..."
        sh """
          docker rmi ${DOCKER_REGISTRY}/${IMAGE_NAME}:${imageTag} || true
          docker image prune -f || true
        """
        cleanWs()
      }
    }
  }
}
