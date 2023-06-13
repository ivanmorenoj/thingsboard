pipeline {
  agent {
    kubernetes {
      inheritFrom 'jenkins-slave'
      yaml '''
      spec:
        containers:
        - name: trufflehog
          image: trufflesecurity/trufflehog:3.34.0
          command:
          - /bin/cat
          tty: true
        - name: trivy
          image: aquasec/trivy:0.41.0
          command:
          - /bin/cat
          tty: true
      '''
    }
  }
  stages {

    stage ('TruffleHog') {
      steps {
        container(name: 'trufflehog') {
          script {
            try{
              sh """
                trufflehog --no-update --only-verified --fail --json filesystem ./ | tee trufflehog-output.json
                
                [[ -z \$(grep '[^[:space:]]' trufflehog-output.json) ]] && rm -v trufflehog-output.json
              """
            } catch (Exception e){
              unstable('Trufflehog Exception')
            }

            if (findFiles(glob: "trufflehog-output.json").length > 0) {
              sh "cat trufflehog-output.json"
            }
          }
        }
      }
    }

    stage('Trivy'){
      steps {
        container(name: 'trivy') {
          script {
            try {
              sh """
                trivy fs --quiet --no-progress --scanners vuln,secret,config --format table ./
                trivy fs --exit-code 1 --severity LOW,MEDIUM,HIGH,CRITICAL --scanners vuln,config \
                  --quiet --format json \
                  --output trivy-findings.json ./
              """
            } catch (Exception e) {
              unstable("Trivy vulnerabilities found!")
            }

            if (findFiles(glob: "trivy-findings.json").length > 0) {
              sh "cat trivy-findings.json"
            }
          }
        }
      }
    }

  }
}

