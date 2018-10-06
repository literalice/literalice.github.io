pipeline {
  agent {
    node {
      label 'hugo'
    }
  }
  stages {
    stage('build') {
      steps {
        script {
          sh "hugo"
        }
      }
      post {
        success {
          archiveArtifacts artifacts: 'docs/**/*', fingerprint: true
        }
      }
    }
    stage('deploy') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject() {
              echo "Using project: ${openshift.project()}"
              sshagent (credentials: ["${openshift.project()}-literalice.github.io"]) {
                sh "git config user.name 'Masatoshi Hayashi'"
                sh "git config user.email 'literalice@monochromeroad.com'"
                sh "git add --all docs"
                sh "git commit -m 'release'"
                sh 'git push origin develop'
              }
            }
          }
        }
      }
    }
  }
}
