// Jenkins pipeline — polls Terracotta-OSS/pr-review-queue for new job files
// and dispatches terracotta-bob-action/.github/workflows/dispatch.yml on GHE.
//
// Prerequisites (configure in Jenkins → Credentials):
//
//   ghes-app-terracotta   Username/password   GitHub App credential:
//                           username = App ID
//                           password = installation token (or private key,
//                                      depending on how your Jenkins GitHub
//                                      App plugin generates the token)
//                         Requires actions:write on
//                           ibm-webmethods/terracotta-bob-action
//
// SCM config for this job:
//   Repository URL : git@github.com:Terracotta-OSS/pr-review-queue.git
//   Credentials    : SSH key or HTTPS credential with read access to pr-review-queue
//   Branch         : */main
//   Poll SCM       : H/2 * * * *    (every ~2 minutes)
//
// Note: no API token is needed for the queue repo. Jenkins checks it out via
// the SCM credential above and reads files directly from the workspace.

pipeline {

  agent { label 'linux' }   // replace with your agent label

  options {
    disableConcurrentBuilds()    // only one poll run at a time
    timeout(time: 10, unit: 'MINUTES')
  }

  environment {
    GHE_API       = 'https://api.github.com'
    DISPATCH_REF  = 'main'
    DISPATCH_WF   = 'dispatch.yml'
    DISPATCH_REPO = 'ibm-webmethods/terracotta-bob-action'
  }

  stages {
    stage('Find new job files') {
      steps {
        script {
          // On the very first build GIT_PREVIOUS_SUCCESSFUL_COMMIT is empty.
          // Fall back to the empty tree SHA so git diff covers the full history.
          def prevCommit = env.GIT_PREVIOUS_SUCCESSFUL_COMMIT?.trim()
            ?: '4b825dc642cb6eb9a060e54bf8d69288fbee4904'

          // Use shell env vars (not Groovy GString interpolation) for the commit
          // SHAs so they are read from the process environment, not embedded in
          // the Groovy string before the sh step runs.
          def diffOutput = sh(
            returnStdout: true,
            script: """
              git diff --name-only --diff-filter=A \
                ${prevCommit} \${GIT_COMMIT} \
                -- jobs/
            """
          ).trim()

           def newFiles = diffOutput
            .split('\n')
            .findAll { it.trim() && !it.trim().endsWith('.gitkeep') }
            .collect { it.trim() }

          // Write the list to a workspace file so the next stage can read it
          // without touching rawBuild internals (keeps us inside the sandbox).
          writeFile file: '.new_job_files', text: newFiles.join('\n')

          if (newFiles.isEmpty()) {
            echo 'No new job files — nothing to dispatch.'
          } else {
            echo "New jobs (${newFiles.size()}):\n${newFiles.join('\n')}"
          }
        }
      }
    }

    stage('Dispatch jobs') {
      steps {
        script {
          def newFiles = readFile('.new_job_files').split('\n').findAll { it.trim() }

          if (newFiles.isEmpty()) {
            echo 'Nothing to dispatch.'
            return
          }

          // usernamePassword binding: APP_ID = app ID (username), GHE_TOKEN = installation token (password).
          // Reference as \${GHE_TOKEN} inside sh strings so Groovy does NOT
          // interpolate it — the shell reads it from the environment and
          // Jenkins masks it correctly in logs.
          withCredentials([
            usernamePassword(
              credentialsId: 'ghes-app-terracotta',
              usernameVariable: 'APP_ID',
              passwordVariable: 'GHE_TOKEN'
            )
          ]) {
            newFiles.each { jobPath ->
              echo "Dispatching: ${jobPath}"

              // readJSON reads from the SCM checkout workspace (queue repo)
              def job = readJSON file: jobPath

              // All workflow_dispatch inputs must be strings
              def inputs = [
                command:            job.command as String,
                target_org:         job.target_org as String,
                target_repo:        job.target_repo as String,
                pr_number:          (job.pr_number as int).toString(),
                actor:              job.actor as String,
                target_github_host: (job.target_github_host ?: 'https://api.github.com') as String,
                max_cost_review:    (job.max_cost_review  ?: 5).toString(),
                max_cost_summary:   (job.max_cost_summary ?: 2).toString(),
                max_cost_triage:    (job.max_cost_triage  ?: 2).toString()
              ]

              def payload = groovy.json.JsonOutput.toJson([
                ref:    env.DISPATCH_REF,
                inputs: inputs
              ])

              // Unique temp files per build tag — safe even if concurrency
              // protection somehow fails
              def payloadFile = ".dispatch_payload_${env.BUILD_TAG}.json"
              def respFile    = ".dispatch_resp_${env.BUILD_TAG}.json"

              writeFile file: payloadFile, text: payload

              // \${GHE_TOKEN} — escaped: shell reads from env, not Groovy string
              def http = sh(
                returnStdout: true,
                script: """
                  curl -s -o ${respFile} -w "%{http_code}" \\
                    -X POST \\
                    -H "Accept: application/vnd.github+json" \\
                    -H "Authorization: Bearer \${GHE_TOKEN}" \\
                    -H "X-GitHub-Api-Version: 2022-11-28" \\
                    "${env.GHE_API}/repos/${env.DISPATCH_REPO}/actions/workflows/${env.DISPATCH_WF}/dispatches" \\
                    -d @${payloadFile}
                """
              ).trim()

              if (http != '204') {
                def resp = readFile(respFile).trim()
                error("workflow_dispatch failed (HTTP ${http}) for ${jobPath}: ${resp}")
              }

              echo "Dispatched ${jobPath} — HTTP ${http}"
            }
          }
        }
      }
    }
  }

  post {
    failure {
      echo 'One or more dispatches failed. Job files remain in the queue repo and will be retried on the next successful build.'
    }
    cleanup {
      // Remove temp files from workspace
      sh "rm -f .new_job_files .dispatch_payload_${env.BUILD_TAG}.json .dispatch_resp_${env.BUILD_TAG}.json || true"
    }
  }
}
