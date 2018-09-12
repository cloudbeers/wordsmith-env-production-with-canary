def label = "wordsmith-deployment-${UUID.randomUUID().toString()}"

def deployment(String releaseKey, String valuesKey) {
    container('helm') {
        for (application in environment.applications) {
            def jenkinsCredentials = []
            def arguments = []
            def idx = 0
            for (credentials in application.credentials) {
                jenkinsCredentials.add(usernamePassword(credentialsId: "${credentials.jenkinsCredentialsId}", passwordVariable: "CREDS_${idx}_PSW", usernameVariable: "CREDS_${idx}_USR"))
                arguments.add("--set ${credentials.helmUsernameParameter}=\$CREDS_${idx}_USR,${credentials.helmPasswordParameter}=\$CREDS_${idx}_PSW")
            }

            if (application[valuesKey]?.trim()) {
                arguments.add "--values ${application[valuesKey]}"
            }
            withCredentials(jenkinsCredentials) {
                try {
                    sh """
                      helm fetch ${application.chart} --version=${application.version}
                      helm upgrade --install ${application[releaseKey]} ${application.chart} --version=${
                        application.version
                    } --namespace ${environment.namespace} --wait ${arguments.join(' ')}
                  """
                } catch (Exception e) {
                    def deploymentIssue = [fields: [
                            project: [key: 'WOR'],
                            summary: "Deployment failure: ${application[releaseKey]}",
                            description: "Please go to ${BUILD_URL} and verify the deployment logs",
                            issuetype: [name: 'Bug']]]

                    jiraResponse = jiraNewIssue issue: deploymentIssue
                    echo "https://jira.beescloud.com/projects/WOR/issues/${jiraResponse.data.key}"
                    throw e
                }
            }
        }
    }
}

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
  properties([disableConcurrentBuilds(), ])
  node (label) {
    def environment
    stage('Load Environment Definition') {
      checkout scm
      environment = readYaml file: 'environment.yaml'
    }
    stage("Repo update") {
        container('helm') {
            sh """
             helm init --client-only
             helm repo add wordsmith https://charts.wordsmith.beescloud.com
             helm repo update
          """
        } // container
    } // stage
    stage("Enable canary"){
        deployment("canaryRelease", "canaryValues")
    } // stage
    stage("Confirm deployment") {
        input 'Confirm Canary deployment?'
    }
    stage("Update real"){
        deployment("release", "values")
        archiveArtifacts artifacts: "*.tgz", fingerprint: true
        def deploymentIssue = [fields: [
                project: [key: 'WOR'],
                summary: "Verify deployment on ${environment.namespace}",
                description: "Please go to ${BUILD_URL} and verify the deployment logs",
                issuetype: [name: 'Task']]]

        jiraResponse = jiraNewIssue issue: deploymentIssue
        echo "Jira verification task created https://jira.beescloud.com/projects/WOR/issues/${jiraResponse.data.key}"
    } // stage
    stage("Disable canary"){
        container('helm'){
            sh """
                helm del --purge ${application[releaseKey]}
            """
        } // container
    } // stage
  } // node
} // podTemplate
