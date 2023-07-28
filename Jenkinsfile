@Library('pipeline-library')
import static dk.stiil.pipeline.Constants.*
import java.util.regex.Pattern

podTemplate(yaml: '''
    apiVersion: v1
    kind: Pod
    spec:
      containers:
      - name: k8s
        image: alpine/k8s:1.27.4
        command:
        - sleep
        args: 
        - 99d
      serviceAccountName: jenkins-tester
      restartPolicy: Never
''') {
TreeMap scmData
String commitMessage
Pattern pattern = Pattern.compile("#[a-zA-Z0-9._-]+\\s([a-zA-Z0-9._-]+)\\s([a-zA-Z0-9._-]+)")
  node(POD_LABEL) {
    stage('checkout SCM') {
      scmData = checkout scm
      commitMessage = sh(returnStdout: true, script: 'git log -1 --pretty=%B').trim()
    }
    container('k8s') {
      stage('Lint charts') {
      
        def files = findFiles() 
        files.each{ f -> 
          if(f.directory && !f.name.startsWith('.') ) {
            sh(script: "helm lint ./${f.name}/")
          }
        }
      }
      def matcher = commitMessage =~ pattern
      if (matcher.find()){
        String component = matcher[0][1]
        String version = matcher[0][1]
        stage('Release Chart') {
          chart = readYaml file: component+"/Chart.yaml"
          if (chart.version == version){
            println "create tag and do stuff"
          }
          else
          {
           error "Missmatch in build version of chart selected for release and specified version"
          }
        }
      }
    }
  }
}
