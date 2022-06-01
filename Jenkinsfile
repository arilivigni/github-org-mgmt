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
    "required_status_checks":{
        "strict":true,
        "contexts":[
          "continuous-integration/jenkins/branch"
        ]
    },
    "enforce_admins":true,
    "required_pull_request_reviews":{
        "dismissal_restrictions":{
          "users":[
              "arilivigni"
          ],
          "teams":[
              "fantastic-four"
          ]
        },
        "dismiss_stale_reviews":true,
        "require_code_owner_reviews":true,
        "required_approving_review_count":2,
        "bypass_pull_request_allowances":{
          "users":[
              "arilivigni"
          ],
          "teams":[
              "fantastic-four"
          ]
        }
    },
    "restrictions":{
        "users":[
          "arilivigni"
        ],
        "teams":[
          "fantastic-four"
        ],
        "apps":[
          "super-ci"
        ]
    },
    "required_linear_history":true,
    "allow_force_pushes":true,
    "allow_deletions":true,
    "block_creations":true,
    "required_conversation_resolution":true
  }"""

  stage("Apply Branch-Protection-Rules") {
    if(env.branch_name && "${branch_name}" == "${master_branch}") {
        withCredentials([string(credentialsId: 'githubToken', variable: 'githubToken')]) {
          echo "organization: ${organization}"
          echo "repository: ${repository}"
          echo "branch_name: ${branch_name}"
          echo "master_branch: ${master_branch}"
          echo "ref_type: ${ref_type}"
          echo "url: ${repository_url}/branches/${repository_default_branch}/protection"
          echo "${githubPayload}"          
          httpRequest(
              contentType: 'APPLICATION_JSON',
              consoleLogResponseBody: true,
              customHeaders: [
                  [maskValue: true, name: 'Authorization', value: "token ${githubToken}"],
                  [name: 'Accept', value: 'application/vnd.github.v3+json']],
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