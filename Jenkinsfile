pipeline {
  agent any

  environment {
    REGISTRY = ""  // Use empty for Minikube
    IMAGE_BLUE = "myapp:blue"
    IMAGE_GREEN = "myapp:green"
  }

  parameters {
    choice(name: 'COLOR', choices: ['blue','green'], description: 'Which deployment to create/update')
    booleanParam(name: 'SWITCH', defaultValue: false, description: 'Switch traffic to this version?')
  }

  stages {
    stage('Build Images') {
      steps {
        script {
          def dir = params.COLOR
          sh """
            docker build -t ${params.COLOR == 'blue' ? IMAGE_BLUE : IMAGE_GREEN} ./${dir}
            minikube image load ${params.COLOR == 'blue' ? IMAGE_BLUE : IMAGE_GREEN}
          """
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        sh """
          kubectl apply -n jenkins -f k8s/${params.COLOR}-deployment.yaml
          kubectl rollout status deployment/myapp-${params.COLOR} -n jenkins
        """
      }
    }

    stage('Traffic Switch') {
      when { expression { params.SWITCH } }
      steps {
        sh """
          kubectl patch svc myapp-svc -n jenkins \
            -p '{"spec":{"selector":{"version":"${params.COLOR}"}}}'
        """
      }
    }
  }

  post {
    always {
      sh """
        echo "=== Service Selector ==="
        kubectl get svc myapp-svc -n jenkins -o jsonpath='{.spec.selector}'
        echo
        echo "=== Deployments ==="
        kubectl get deployments -n jenkins -l app=myapp
      """
    }
    success {
      echo "Deployment pipeline completed successfully."
    }
    failure {
      echo "Pipeline failed. Inspect logs for details."
    }
  }
}
