pipeline {
  agent any

  stages {
    stage('Trigger CoReview external review') {
      when {
        // Chỉ chạy khi build từ Pull Request (CHANGE_ID có giá trị)
        expression { env.CHANGE_ID?.trim() }
      }
      steps {
        withCredentials([
          string(credentialsId: 'coreview-api-url', variable: 'COREVIEW_API_URL'),
          string(credentialsId: 'coreview-trigger-token', variable: 'COREVIEW_TRIGGER_TOKEN'),
        ]) {
          script {
            // Parse org/repo từ GIT_URL
            // Ví dụ: https://github.com/myorg/myrepo.git
            def gitUrl = env.GIT_URL ?: ''
            def matcher = (gitUrl =~ /github\.com[:/](.+?)(?:\.git)?$/)
            if (!matcher.find()) {
              error("Cannot parse repository from GIT_URL: ${gitUrl}")
            }
            def repository = matcher.group(1)

            def prNumber = env.CHANGE_ID.toInteger()
            def title = env.CHANGE_TITLE ?: ''
            def headRef = env.CHANGE_BRANCH ?: ''

            // CoReview API: synchronize vẫn map action "opened"
            def triggerAction = 'opened'

            def payload = groovy.json.JsonOutput.toJson([
              provider     : 'github',
              repository   : repository,
              pull_request : [
                number  : prNumber,
                title   : title,
                head_ref: headRef,
              ],
              action       : triggerAction,
              source       : 'jenkins',
            ])

            echo "Triggering CoReview for ${repository}#${prNumber}"

            def response = sh(
              script: """
                set -euo pipefail
                HTTP_CODE=\$(curl -sS -o /tmp/coreview-response.json -w '%{http_code}' \\
                  -X POST "\${COREVIEW_API_URL}/api/v1/reviews/external/trigger" \\
                  -H "Authorization: Bearer \${COREVIEW_TRIGGER_TOKEN}" \\
                  -H "Content-Type: application/json" \\
                  -d '${payload.replace("'", "'\\''")}')
                echo "HTTP \${HTTP_CODE}"
                cat /tmp/coreview-response.json
                test "\${HTTP_CODE}" -ge 200 && test "\${HTTP_CODE}" -lt 300
              """,
              returnStdout: true
            )
            echo response
          }
        }
      }
    }
  }
}
