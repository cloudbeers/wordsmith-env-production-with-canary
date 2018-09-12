def label = "wordsmith-deployment-${UUID.randomUUID().toString()}"
podTemplate(label: label, yaml: """
apiVersion: v1
kind: Pod
spec:
    containers:
    - name: jnlp
    - name: helm
      image: devth/helm
      command:
      - cat
      tty: true
    - name: kubectl
      image: lachlanevenson/k8s-kubectl:v1.10.7
      command:
      - cat
      tty: true
    - name: curl
      image: appropriate/curl
      command:
      - cat
      tty: true
    - name: jdk
      image: openjdk:8-jdk
      command:
      - cat
      tty: true
"""
    ) {
  properties([disableConcurrentBuilds()])
  node (label) {
    def environment
    stage('Load Environment Definition') {
      checkout scm
      environment = readYaml file: 'environment.yaml'
    }
    stage('Install Helm Charts') {
        container('helm') {
          sh """
             helm init --client-only
             helm repo add wordsmith https://charts.wordsmith.beescloud.com
             helm repo update
          """
          for (application in environment.applications) {
              def jenkinsCredentials = []
              def arguments = []
              def idx=0
              for (credentials in application.credentials) {
                jenkinsCredentials.add(usernamePassword(credentialsId: "${credentials.jenkinsCredentialsId}", passwordVariable: "CREDS_${idx}_PSW", usernameVariable: "CREDS_${idx}_USR"))
                arguments.add("--set ${credentials.helmUsernameParameter}=\$CREDS_${idx}_USR,${credentials.helmPasswordParameter}=\$CREDS_${idx}_PSW")
              }
              
              if (application.values?.trim()) {
                  arguments.add "--values ${application.values}"
              }
              withCredentials (jenkinsCredentials) {
                try {
                  sh """
                      helm fetch ${application.chart} --version=${application.version}
                      helm upgrade --install ${application.release} ${application.chart} --version=${application.version} --namespace ${environment.namespace} --wait ${arguments.join(' ')}
                  """
                } catch (Exception e) {
                  def deploymentIssue = [fields: [
                               project: [key: 'WOR'],
                               summary: "Deployment failure: ${application.release}",
                               description: "Please go to ${BUILD_URL} and verify the deployment logs",
                               issuetype: [name: 'Bug']]]

                  jiraResponse = jiraNewIssue issue: deploymentIssue
                  echo "https://jira.beescloud.com/projects/WOR/issues/${jiraResponse.data.key}"
                  throw e
                }
              }
          }
          archiveArtifacts artifacts: "*.tgz", fingerprint: true
          def deploymentIssue = [fields: [
             project: [key: 'WOR'],
             summary: "Verify deployment on ${environment.namespace}",
             description: "Please go to ${BUILD_URL} and verify the deployment logs",
             issuetype: [name: 'Task']]]

          jiraResponse = jiraNewIssue issue: deploymentIssue
          echo "Jira verification task created https://jira.beescloud.com/projects/WOR/issues/${jiraResponse.data.key}"
        } // container
    } // stage
  } // node
} // podTemplate
