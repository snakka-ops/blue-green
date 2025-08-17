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
    - name: dockersock
      mountPath: /var/run/docker.sock
    - name: workspace
      mountPath: /home/jenkins/agent
  - name: kubectl
    image: bitnami/kubectl:1.29
    command:
    - cat
    tty: true
    volumeMounts:
    - name: workspace
      mountPath: /home/jenkins/agent
  - name: jnlp
    image: jenkins/inbound-agent:latest
    args: ['\$(JENKINS_SECRET)', '\$(JENKINS_NAME)']
    volumeMounts:
    - name: workspace
      mountPath: /home/jenkins/agent
  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock
      type: Socket
  - name: workspace
    emptyDir: {}
"""
    }
  }

  environment {
    IMAGE_BLUE  = "myapp:blue"
    IMAGE_GREEN = "myapp:green"
    NAMESPACE   = "jenkins"
  }

  parameters {
    choice(name: 'COLOR', choices: ['blue','green'], description: 'Which color to deploy')
    booleanParam(name: 'SWITCH', defaultValue: false, description: 'Switch service traffic to this color?')
  }

  stages {
    stage('Build Image') {
      steps {
        container('docker') {
          script {
            sh """
              docker build -t ${params.COLOR == 'blue' ? IMAGE_BLUE : IMAGE_GREEN} ./${params.COLOR}
            """
          }
        }
      }
    }

    stage('Deploy') {
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
            kubectl patch svc myapp-svc -n ${NAMESPACE} \
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
          echo "Service selector now:" 
          kubectl get svc myapp-svc -n ${NAMESPACE} -o jsonpath='{.spec.selector}'
          echo
          echo "Current deployments:"
          kubectl get deployments -n ${NAMESPACE} -l app=myapp
        """
      }
    }
    success {
      echo "✅ Pipeline completed successfully."
    }
    failure {
      echo "❌ Pipeline failed; check logs for details."
    }
  }
}
