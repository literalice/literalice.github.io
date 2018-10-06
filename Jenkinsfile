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
    }
  }
}

