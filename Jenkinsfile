pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: bluegreen-pipeline
spec:
  serviceAccountName: jenkins-agent
  containers:
  - name: docker
    image: docker:24.0.7-cli
    command:
      - cat
    tty: true
    volumeMounts:
      - name: docker-sock
        mountPath: /var/run/docker.sock
  - name: kubectl
    image: lachlanevenson/k8s-kubectl:latest
    command:
      - cat
    tty: true
  - name: jnlp
    image: jenkins/inbound-agent:latest
    args:
      - "\$(JENKINS_SECRET)"
      - "\$(JENKINS_NAME)"
    tty: true
  volumes:
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
        type: Socket
"""
    }
  }

  environment {
    NAMESPACE    = "jenkins"
    IMAGE_BLUE   = "myapp:blue"
    IMAGE_GREEN  = "myapp:green"
  }

  parameters {
    choice(name: 'COLOR', choices: ['blue','green'], description: 'Color to deploy')
    booleanParam(name: 'SWITCH', defaultValue: false, description: 'Switch service to this version?')
  }

  stages {
    stage('Build Image') {
      steps {
        container('docker') {
          sh """
            cd ${params.COLOR}
            docker build -t ${params.COLOR == 'blue' ? IMAGE_BLUE : IMAGE_GREEN} .
          """
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        container('kubectl') {
          sh """
            kubectl apply -n ${NAMESPACE} -f k8s/${params.COLOR}-deployment.yaml
            kubectl rollout status deployment/myapp-${params.COLOR} -n ${NAMESPACE} --timeout=300s
          """
        }
      }
    }

    stage('Switch Traffic') {
      when { expression { params.SWITCH } }
      steps {
        container('kubectl') {
          sh """
            kubectl patch svc myapp-svc -n ${NAMESPACE} \\
              -p '{\"spec\":{\"selector\":{\"version\":\"${params.COLOR}\"}}}'
          """
        }
      }
    }
  }

  post {
    always {
      container('kubectl') {
        sh """
          echo '=== Service Selector ==='
          kubectl get svc myapp-svc -n ${NAMESPACE} -o jsonpath='{.spec.selector}'
          echo
          echo '=== Current Deployments ==='
          kubectl get deployments -n ${NAMESPACE} -l app=myapp
        """
      }
    }
    success {
      echo "✅ Pipeline completed successfully."
    }
    failure {
      echo "❌ Pipeline failed; please review the logs above."
    }
  }
}
