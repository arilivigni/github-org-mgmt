node {
  properties([
    [$class: 'BuildDiscarderProperty',
      strategy: [$class: 'LogRotator',
        artifactDaysToKeepStr: '',
        artifactNumToKeepStr: '',
        daysToKeepStr: '',
        numToKeepStr: '5']
    ],
    pipelineTriggers([
      [$class: 'GenericTrigger',
        genericVariables: [
          [expressionType: 'JSONPath', key: 'repository', value: '$.repository'],
          [expressionType: 'JSONPath', key: 'organization', value: '$.organization'],
          [expressionType: 'JSONPath', key: 'sender', value: '$.sender'],
          [expressionType: 'JSONPath', key: 'ref_type', value: '$.ref_type'],
          [expressionType: 'JSONPath', key: 'master_branch', value: '$.master_branch', defaultValue: 'main'],
          [expressionType: 'JSONPath', key: 'branch_name', value: '$.ref']
        ],
        tokenCredentialId: 'githubToken',
        regexpFilterText: '',
        regexpFilterExpression: ''
      ]
    ])
  ])

  def githubPayload = """{
    "required_status_checks": {
        "required_approving_review_count": 2,
        "strict": true,
        "contexts": [
            "continuous-integration/jenkins/branch"
        ]
  }"""

  stage("Apply Branch-Protection-Rules") {
    if(env.branch_name && "${branch_name}" == "${master_branch}") {
        withCredentials([string(credentialsId: 'githubToken', variable: 'githubToken')]) {
          httpRequest(
              contentType: 'APPLICATION_JSON',
              consoleLogResponseBody: true,
              customHeaders: [
                  [maskValue: true, name: 'Authorization', value: "token ${githubToken}"],
                  [name: 'Accept', value: 'application/vnd.github.loki-preview']],
              httpMode: 'PUT',
              ignoreSslErrors: true,
              requestBody: githubPayload,
              responseHandle: 'NONE',
              url: "${repository_url}/branches/${repository_default_branch}/protection")
        }
    } else {
        sh(name: "Skip", script: 'echo "Move along, nothing to see here"')
    }
  }
}