pipeline {
  agent {
    kubernetes {
      inheritFrom 'jenkins-slave'
      yaml '''
      spec:
        containers:
        - name: alpine
          image: alpine
          command:
          - /bin/cat
          tty: true
        - name: trufflehog
          image: trufflesecurity/trufflehog:3.40.0
          command:
          - /bin/cat
          tty: true
        - name: trivy
          image: aquasec/trivy:0.42.1
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

            if (fileExists('trufflehog-output.json')) {
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

            if (fileExists("trivy-findings.json")) {
              sh "cat trivy-findings.json"
            }
          }
        }
      }
    }

    stage('Upload to DefectDojo') {
      steps {
        container(name: 'alpine') {
          sh label: "Intall required packages",
          script: """
            apk add --no-cache jq curl 
          """

          script {
            if (fileExists('trufflehog-output.json')) {
              echo "Trufflehog vulnerabilities found!!"
              sh "cat trufflehog-output.json"
            }

            if (fileExists("trivy-findings.json")) {
              echo "Trivy vulnerabilities found!!"
              sh "cat trivy-findings.json"
            }
          }
        }
      }
    }

  }
}

