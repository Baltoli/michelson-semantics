pipeline {
  agent {
    dockerfile {
      label 'docker'
      additionalBuildArgs '--build-arg K_COMMIT=$(cd ext/k && git tag --points-at HEAD | cut --characters=2-) --build-arg USER_ID=$(id -u) --build-arg GROUP_ID=$(id -g)'
    }
  }
  options { ansiColor('xterm') }
  environment {
    GITHUB_TOKEN     = credentials('rv-jenkins-access-token')
    LONG_REV = """${sh(returnStdout: true, script: 'git rev-parse HEAD').trim()}"""
  }
  stages {
    stage('Init title') {
      when { changeRequest() }
      steps { script { currentBuild.displayName = "PR ${env.CHANGE_ID}: ${env.CHANGE_TITLE}" } }
    }
    stage('Build') {
      parallel {
        stage('K')      { steps { sh 'make build-k -j8 RELEASE=true' } }
      }
    }
    stage('Cross Test') {
      stages {
        stage('Build Tezos')      { steps { sh 'make deps-tezos'        } }
        stage('Build Compat')     { steps { sh 'make build-compat -j8 RELEASE=true' } }
        stage('Cross-Validation') { steps { sh 'make test-cross    -j8' } }
      }
    }
  }
}
