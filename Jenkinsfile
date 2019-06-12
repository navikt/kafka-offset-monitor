pipeline {
  agent any
  environment {
    DEPLOYMENT = readYaml(file: './nais.yaml')
    APPLICATION_NAME = "${DEPLOYMENT.metadata.name}"
    ZONE = "${DEPLOYMENT.metadata.annotations.zone}"
    NAMESPACE = "${DEPLOYMENT.metadata.namespace}"
    VERSION = sh(label: 'Get git sha1 as version', script: 'git rev-parse --short HEAD', returnStdout: true).trim()
  }

  stages {
    stage('Build') {
      steps {
        sh './gradlew build'
      }

      post {
        always {
          publishHTML target: [
            allowMissing: true,
            alwaysLinkToLastBuild: false,
            keepAll: true,
            reportDir: 'build/reports/tests/test',
            reportFiles: 'index.html',
            reportName: 'Test reports'
          ]

          junit 'build/test-results/test/*.xml'
        }
      }
    }

    stage('Deploy to non-production') {
      when { branch 'master' }

      environment {
        DOCKER_REPO = 'repo.adeo.no:5443'
        DOCKER_IMAGE_VERSION = '${DOCKER_REPO}/${APPLICATION_NAME}:${VERSION}'
      }

      steps {
        withDockerRegistry(
          credentialsId: 'repo.adeo.no',
          url: "https://${DOCKER_REPO}"
        ) {
          sh label: 'Build and push Docker image', script: """
            docker build . --pull -t ${DOCKER_IMAGE_VERSION}
            docker push ${DOCKER_IMAGE_VERSION}
          """
        }

        sh label: 'Prepare service contract', script: """
            sed 's/latest/${VERSION}/' nais.yaml | tee nais-deployed.yaml
        """

        sh label: 'Deploy to non-production', script: """
          kubectl config use-context dev-${env.ZONE}
          kubectl apply -n ${env.NAMESPACE} -f nais-deployed.yaml --wait
          kubectl rollout status -w deployment/${APPLICATION_NAME}
        """
      }

      post {
        success {
          archiveArtifacts artifacts: 'nais.yaml', fingerprint: true
        }
      }
    }

    stage("Deploy to production") {
      when { branch 'master' }

      steps {
        sh label: 'Deploy to production', script: """
          kubectl config use-context prod-${env.ZONE}
          kubectl apply -n ${env.NAMESPACE} -f nais-deployed.yaml --wait
          kubectl rollout status -w deployment/${APPLICATION_NAME}
        """
      }

    }

  }
}
