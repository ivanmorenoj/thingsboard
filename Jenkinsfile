pipeline {
  agent {
    kubernetes {
      inheritFrom 'jenkins-slave'
      yaml '''
      spec:
        containers:
        - name: dockle
          image: goodwithtech/dockle:v0.4.9
          command:
          - "cat"
          tty: true
      '''
    }
  }
  stages {

    stage ('Next stages...') {
      steps {
        sh """
          echo 'Executing Next stages...'
        """
      }
    }

  }
}

