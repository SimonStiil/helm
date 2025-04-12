@Library('pipeline-library')
import static dk.stiil.pipeline.Constants.*
import java.util.regex.Pattern
// library located at https://github.com/SimonStiil/pipeline-library

// NonCPS is due to matcher not being serializeable and thowing exceptions.
@NonCPS
def releaseMessage(String commitMessage){
    // get information from commit message. 
    Pattern pattern = Pattern.compile("#([a-zA-Z0-9._-]+)\\s([a-zA-Z0-9._-]+)\\s([a-zA-Z0-9._-]+)")
    def matcher = commitMessage =~ pattern
    // will get the first 3 values and return them in map.
    if (matcher.find()){
        return [action: matcher[0][1], component: matcher[0][2], version:matcher[0][3]]
    }
    // TODO: could be made more generic
    return null
}

// Image: alpine/k8s used for its helm functionality
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
      /* Return statement from "checkout scm" contains a lot of great information:
       * GIT_AUTHOR_EMAIL, GIT_AUTHOR_NAME, GIT_BRANCH, GIT_COMMIT, GIT_COMMITTER_EMAIL
       * GIT_COMMITTER_NAME, GIT_PREVIOUS_COMMIT, GIT_PREVIOUS_SUCCESSFUL_COMMIT,
       * GIT_URL
       */ 
      // Save scmData & commitMessage for later
      commitMessage = sh(returnStdout: true, script: 'git log -1 --pretty=%B').trim()
      // setup username and email for jenkins user 
      sh(script: "git config --global user.email \"130985998+SimonStiil@users.noreply.github.com\" && git config --global user.name \"Jenkins\"")
    }
    container('k8s') {
      stage('Lint charts') {
        // Start by listing all files
        def files = findFiles() 
        files.each{ f -> 
          // Work only with directories that does not start with .
          if(f.directory && !f.name.startsWith('.') ) {
            // Lint the directory as a chart
            sh(script: "helm lint ./${f.name}/")
          }
        }
      }
    }
    // TODO: Was lazy and did not want to move the brackets.
    // Could be done only when branchname is main
    release = releaseMessage(commitMessage)
    if (BRANCH_NAME == "main" && release != null && release.action == "release"){
      // If we are a release from the main branch
      def tagName = release.component + "-" + release.version
      // Find the repo name from the git URL. TODO: Would love something generic for that
      def orgRepo = scmGetOrgRepo scmData.GIT_URL
      def tagHash = ""
      // Are we good?
      stage('Pre Release checks') {
        echo "Chart: " + release.component
        // Reading the chart to match with release 
        chart = readYaml file: release.component+"/Chart.yaml"
        if (chart.version != release.version){
          error "Missmatch in build version of chart selected for release ("+chart.version+") and specified version ("+release.version+")"
        } else {
          echo "Char version matching release cersion: " + chart.version
        }
        // Reading from github if the tag already exists
        def res = sh(script: "git ls-remote origin refs/tags/"+tagName, returnStdout: true) // --exit-code to get exitcode
        tagExists = (! res.trim())
        if (res.trim()){
          warning "Release tag already exists"
          tagHash = res.trim().split("\\s+")[0]
          // tag exists. Is is because the tag is for this commit?
          if (tagHash != scmData.GIT_COMMIT){
            error "Yag is not this commit (" +scmData.GIT_COMMIT+ ") but set to ( "+ tagHash + ")"
          }
        } else {
          echo "Tag does not already exist"
        }
        // So the tag does not exist. But is there are releasy by this name anyway?
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
        // Package the chart specified in the release message
          sh(script: "helm package ./${release.component}/")
        }
        // Will always create a zip as "[chart-name]-[chart-version].tgz"
        def packageName = tagName+".tgz"
        stage('Release Chart') {
          // Create a release on github and upload the package to that release returning the download URL
          downloadURL = githubReleaseFlow repository: orgRepo.repoName, 
                                          release: tagName,
                                          fileName: packageName,
                                          commit: scmData.GIT_COMMIT
         
        }
        // Removing the package name from the download URL as it will be appended by "helm repo index"
        downloadURL = downloadURL.replace(packageName,"")
      }
      stage('Update Index') {
        // create subdir to work with the index branch of this repository
        dir(orgRepo.repoName){
          // Checkout index branch. Dould just be git commands, but didn't have the withCredentials figured out
          git branch: 'index', changelog: false, credentialsId: 'github-login-secret', poll: false, url: "https://github.com/${orgRepo.fullName}.git"
        }
        container('k8s') {
          // Copy index file to working directory, 
          // do helm repo index and append to exisisting index, 
          // copy index back to index branch directory 
          sh(script: "cp ${orgRepo.repoName}/index.yaml . && helm repo index --url ${downloadURL} --merge index.yaml . && cp index.yaml ${orgRepo.repoName}/")
        }
        dir(orgRepo.repoName){
          // with github credentials add, commit and push this change to the index branch
          withCredentials([gitUsernamePassword(credentialsId: 'github-login-secret', gitToolName: 'Default')]) {
            sh(script: "git add index.yaml && git commit -m \"release ${tagName}\" && git push --set-upstream origin index")
          }
        }
      }
    }
  }
}
