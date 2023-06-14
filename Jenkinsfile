pipeline {
  agent {
    kubernetes {
      inheritFrom 'jenkins-slave'
      yaml '''
      spec:
        containers:
        - name: debian
          image: debian
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

  environment {
    DEFECTDOJO_API_KEY = credentials('defectdojo-api-token')
    DEFECTDOJO_HOST = credentials('defectdojo-host')

    DEFECTDOJO_ENGAGEMENT_PERIOD='7'
    DEFECTDOJO_ENGAGEMENT_STATUS='Not Started'
    DEFECTDOJO_ENGAGEMENT_BUILD_SERVER='null'
    DEFECTDOJO_ENGAGEMENT_SOURCE_CODE_MANAGEMENT_SERVER='null'
    DEFECTDOJO_ENGAGEMENT_ORCHESTRATION_ENGINE='null'
    DEFECTDOJO_ENGAGEMENT_DEDUPLICATION_ON_ENGAGEMENT='false'
    DEFECTDOJO_ENGAGEMENT_THREAT_MODEL='true'
    DEFECTDOJO_ENGAGEMENT_API_TEST='true'
    DEFECTDOJO_ENGAGEMENT_PEN_TEST='true'
    DEFECTDOJO_ENGAGEMENT_CHECK_LIST='true'
    DEFECTDOJO_NOT_ON_MASTER='false'
    DEFECTDOJO_SCAN_MINIMUM_SEVERITY='Info'
    DEFECTDOJO_SCAN_ACTIVE='true'
    DEFECTDOJO_SCAN_VERIFIED='true'
    DEFECTDOJO_SCAN_CLOSE_OLD_FINDINGS='true'
    DEFECTDOJO_SCAN_PUSH_TO_JIRA='false'
    DEFECTDOJO_SCAN_ENVIRONMENT='Default'
    DEFECTDOJO_ANCHORE_DISABLE='false'
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
        container(name: 'debian') {
          sh label: "Intall required packages",
          script: """
            apt-get update
            apt-get install -y jq curl
          """

          script {

            env.TODAY = sh (script: "date +%Y-%m-%d", returnStdout: true).trim()
            env.ENDDAY = sh (script: "date -d '+7 days' +%Y-%m-%d", returnStdout: true).trim()
            

            if (fileExists('trufflehog-output.json')) {
              echo "Trufflehog vulnerabilities found!!"
              echo "Upload to DefectDojo"

              sh label: "Create an engagement",
              script: """
                curl --fail --location --request POST "${DEFECDOJO_HOST}/api/v2/engagements/" \
                  --header "Authorization: Token ${DEFECTDOJO_API_TOKEN}" \
                  --header 'Content-Type: application/json' \
                    --data-raw "{
                      \"tags\": [\"JenkinsCI\"],
                      \"name\": \"#${BUILD_DISPLAY_NAME}\",
                      \"description\": \"test\",
                      \"version\": \"test1.1\",
                      \"first_contacted\": \"${TODAY}\",
                      \"target_start\": \"${TODAY}\",
                      \"target_end\": \"${ENDDAY}\",
                      \"reason\": \"string\",
                      \"threat_model\": \"${DEFECTDOJO_ENGAGEMENT_THREAT_MODEL}\",
                      \"api_test\": \"${DEFECTDOJO_ENGAGEMENT_API_TEST}\",
                      \"pen_test\": \"${DEFECTDOJO_ENGAGEMENT_PEN_TEST}\",
                      \"check_list\": \"${DEFECTDOJO_ENGAGEMENT_CHECK_LIST}\",
                      \"status\": \"${DEFECTDOJO_ENGAGEMENT_STATUS}\",
                      \"engagement_type\": \"CI/CD\",
                      \"product\": \"10\",
                      \"deduplication_on_engagement\": \"${DEFECTDOJO_ENGAGEMENT_DEDUPLICATION_ON_ENGAGEMENT}\",
                      \"build_server\": ${DEFECTDOJO_ENGAGEMENT_BUILD_SERVER},
                      \"source_code_management_server\": ${DEFECTDOJO_ENGAGEMENT_SOURCE_CODE_MANAGEMENT_SERVER},
                      \"orchestration_engine\": ${DEFECTDOJO_ENGAGEMENT_ORCHESTRATION_ENGINE}
                    }" -o engagement-response.json
              """

              env.ENGAGEMENT_ID = sh (script: """
                jq -r . '.id' engagement-response.json
                """, returnStdout: true).trim()

              sh label: "Upload artifact",
              script: """
                curl --fail --location --request POST "${DEFECTDOJO_HOST}/api/v2/import-scan/" \
                  --header "Authorization: Token ${DEFECTDOJO_API_TOKEN}" \
                  --form "scan_date=\"${TODAY}\"" \
                  --form "minimum_severity=\"${DEFECTDOJO_SCAN_MINIMUM_SEVERITY}\"" \
                  --form "active=\"${DEFECTDOJO_SCAN_ACTIVE}\"" \
                  --form "verified=\"${DEFECTDOJO_SCAN_VERIFIED}\"" \
                  --form "scan_type=\"Trufflehog3 Scan\"" \
                  --form "engagement=\"${ENGAGEMENT_ID}\"" \
                  --form "file=@trufflehog-output.json" \
                  --form "test_type=\"${DEFECTDOJO_SCAN_TEST_TYPE}\"" \
                  --form "environment=\"${DEFECTDOJO_SCAN_ENVIRONMENT}\""
              """
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

