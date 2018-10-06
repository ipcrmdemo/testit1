import groovy.json.JsonOutput

/**
 * Notify the Atomist services about the status of a build based from a
 * git repository.
 */
def notifyAtomist(
  String workspaceIds,
  String buildStatus,
  String gitUrl,
  String gitBranch,
  String gitCommitSha,
  String buildPhase="FINALIZED"
) {
    if (!workspaceIds) {
        echo 'No Atomist workspace IDs, not sending build notification'
        return
    }

    def payload = JsonOutput.toJson(
        [
            name: env.JOB_NAME,
            duration: currentBuild.duration,
            build: [
                number: env.BUILD_NUMBER,
                phase: buildPhase,
                status: buildStatus,
                full_url: env.BUILD_URL,
                scm: [
                    url: gitUrl,
                    branch: gitBranch,
                    commit: gitCommitSha
                ]
            ]
        ]
    )
    workspaceIds.split(',').each { workspaceId ->
        String endpoint = "https://webhook.atomist.com/atomist/jenkins/teams/${workspaceId}"
        sh "curl --silent -X POST -H 'Content-Type: application/json' -d '${payload}' ${endpoint}"
    }
}


node() {
  withCredentials([[$class: 'StringBinding', credentialsId: 'atomist-workspace',
                variable: 'ATOMIST_WORKSPACES']]) {

      try {
          final scmVars = checkout(scm)
          def url = sh(returnStdout: true, script: 'git config remote.origin.url').trim()
          echo "scmVars: ${scmVars}"
          echo "scmVars.GIT_COMMIT: ${scmVars.GIT_COMMIT}"
          echo "scmVars.GIT_BRANCH: ${scmVars.GIT_BRANCH}"
          echo "url: ${url}"
    
          stage('Notify') {
            echo 'Sending build start...'
            notifyAtomist(
              ATOMIST_WORKSPACES,
              'STARTED',
              url,
              scmVars.GIT_BRANCH,
              scmVars.GIT_COMMIT,
              'STARTED'
            )
          }

          withMaven(maven: 'default') {
              stage('Build, Test, and Package') {
                echo 'Building, testing, and packaging...'
                sh "mvn clean package"
              }
          }

          currentBuild.result = 'SUCCESS'
          notifyAtomist(
            ATOMIST_WORKSPACES,
            currentBuild.result,
            url,
            scmVars.GIT_BRANCH,
            scmVars.GIT_COMMIT
          )

      } catch (Exception err) {

        currentBuild.result = 'FAILURE'
        notifyAtomist(
          ATOMIST_WORKSPACES,
          currentBuild.result,
          url,
          scmVars.GIT_BRANCH,
          scmVars.GIT_COMMIT
        )

      }
  }
}
