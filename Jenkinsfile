@Library('pipeline-library')
import static dk.stiil.pipeline.Constants.*
import java.util.regex.Pattern

@NonCPS
def releaseMessage(String commitMessage){
    Pattern pattern = Pattern.compile("#([a-zA-Z0-9._-]+)\\s([a-zA-Z0-9._-]+)\\s([a-zA-Z0-9._-]+)")
    def matcher = commitMessage =~ pattern
    if (matcher.find()){
        return [action: matcher[0][1], component: matcher[0][2], version:matcher[0][3]]
    }
    return null
}

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
  node(POD_LABEL) {
    stage('checkout SCM') {
      scmData = checkout scm
      /* GIT_AUTHOR_EMAIL, GIT_AUTHOR_NAME, GIT_BRANCH, GIT_COMMIT, GIT_COMMITTER_EMAIL
       * GIT_COMMITTER_NAME, GIT_PREVIOUS_COMMIT, GIT_PREVIOUS_SUCCESSFUL_COMMIT,
       * GIT_URL
       */ 
      commitMessage = sh(returnStdout: true, script: 'git log -1 --pretty=%B').trim()
      sh(script: "git config --global user.email \"robot@stiil.dk\" && git config --global user.name \"Jenkins\"")
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
    }
    release = releaseMessage(commitMessage)
    if (BRANCH_NAME == "main" && release != null && release.action == "release"){
      def tagName = release.component + "-" + release.version
      def orgRepo = scmGetOrgRepo scmData.GIT_URL
      def tagHash = ""
      stage('Pre Release checks') {
        echo "Chart: " + release.component
        chart = readYaml file: release.component+"/Chart.yaml"
        if (chart.version != release.version){
          error "Missmatch in build version of chart selected for release ("+chart.version+") and specified version ("+release.version+")"
        } else {
          echo "Char version matching release cersion: " + chart.version
        }
        def res = sh(script: "git ls-remote origin refs/tags/"+tagName, returnStdout: true) // --exit-code to get exitcode
        tagExists = (! res.trim())
        if (res.trim()){
          warning "Release tag already exists"
          tagHash = res.trim().split("\\s+")[0]
          if (tagHash != scmData.GIT_COMMIT){
            error "Yag is not this commit (" +scmData.GIT_COMMIT+ ") but set to ( "+ tagHash + ")"
          }
        } else {
          echo "Tag does not already exist"
        }
        def resultInfo = githubGetRelease repository: orgRepo.repoName, release: tagName
        if (resultInfo.status != 404) {
          error "Release already exists on github"
        } else {
          echo "Release does not already exist on github"
        }
      }
      def downloadURL
      container('k8s') {
        stage('Package Chart') {
          sh(script: "helm package ./${release.component}/")
        }
        def packageName = tagName+".tgz"
        stage('Release Chart') {
          downloadURL = githubReleaseFlow repository: orgRepo.repoName, 
                                          release: tagName,
                                          fileName: packageName,
                                          commit: scmData.GIT_COMMIT
         
        }
        downloadURL = downloadURL.replace(packageName,"")
      }
      stage('Update Index') {
        dir(orgRepo.repoName){
          git branch: 'index', changelog: false, credentialsId: 'github-login-secret', poll: false, url: "https://github.com/${orgRepo.fullName}.git"
        }
        container('k8s') {
          sh(script: "cp ${orgRepo.repoName}/index.yaml . && helm repo index --url ${downloadURL} --merge index.yaml . && cp index.yaml ${orgRepo.repoName}/")
        }
        dir(orgRepo.repoName){
          withCredentials([gitUsernamePassword(credentialsId: 'github-login-secret', gitToolName: 'Default')]) {
            sh(script: "git add index.yaml && git commit -m \"release ${tagName}\" && git push --set-upstream origin index")
          }
        }
      }
    }
  }
}
