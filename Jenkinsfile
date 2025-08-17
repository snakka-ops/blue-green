pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    some-label: bluegreen
spec:
  containers:
  - name: docker
    image: docker:24.0.7-cli
    command:
    - cat
    tty: true
    volumeMounts:
    - name: var-run-docker
      mountPath: /var/run/docker.sock
  - name: kubectl
    image: bitnami/kubectl:1.29
    command:
    - cat
    tty: true
  volumes:
  - name: var-run-docker
    hostPath:
      path: /var/run/docker.sock
      type: Socket
"""
    }
  }

  environment {
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
        container('docker') {
          script {
            def dir = params.COLOR
            sh """
              docker build -t ${params.COLOR == 'blue' ? IMAGE_BLUE : IMAGE_GREEN} ./${dir}
            """
          }
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        container('kubectl') {
          sh """
            kubectl apply -n jenkins -f k8s/${params.COLOR}-deployment.yaml
            kubectl rollout status deployment/myapp-${params.COLOR} -n jenkins
          """
        }
      }
    }

    stage('Traffic Switch') {
      when { expression { params.SWITCH } }
      steps {
        container('kubectl') {
          sh """
            kubectl patch svc myapp-svc -n jenkins \
              -p '{"spec":{"selector":{"version":"${params.COLOR}"}}}'
          """
        }
      }
    }
  }

  post {
    always {
      container('kubectl') {
        sh """
          echo "=== Service Selector ==="
          kubectl get svc myapp-svc -n jenkins -o jsonpath='{.spec.selector}'
          echo
          echo "=== Deployments ==="
          kubectl get deployments -n jenkins -l app=myapp
        """
      }
    }
    success {
      echo "Deployment pipeline completed successfully."
    }
    failure {
      echo "Pipeline failed. Inspect logs for details."
    }
  }
}
