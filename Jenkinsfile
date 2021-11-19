/*
** Variables.
*/
properties([buildDiscarder(logRotator(numToKeepStr: '50'))])
def serie = '20.10'
def maintenanceBranch = "${serie}.x"
if (env.BRANCH_NAME.startsWith('release-')) {
  env.BUILD = 'RELEASE'
} else if ((env.BRANCH_NAME == 'master') || (env.BRANCH_NAME == maintenanceBranch)) {
  env.BUILD = 'REFERENCE'
} else {
  env.BUILD = 'CI'
}
def featureFiles = []

/*
** Pipeline code.
*/
stage('Source') {
  node {
    sh 'setup_centreon_build.sh'
    env.WIDGET = 'tactical-overview'
    dir("centreon-widget-${env.WIDGET}") {
      checkout scm
    }
    sh "./centreon-build/jobs/widgets/${serie}/widget-source.sh"
    source = readProperties file: 'source.properties'
    env.PROJECT = "${source.PROJECT}"
    env.VERSION = "${source.VERSION}"
    env.RELEASE = "${source.RELEASE}"
    env.SUMMARY = "${source.SUMMARY}"
    publishHTML([
      allowMissing: false,
      keepAll: true,
      reportDir: 'summary',
      reportFiles: 'index.html',
      reportName: 'Centreon Build Artifacts',
      reportTitles: ''
    ])
  }
}

try {
  stage('Package') {
    parallel 'centos7': {
      node {
        sh 'setup_centreon_build.sh'
        sh "./centreon-build/jobs/widgets/${serie}/widget-package.sh centos7"
      }
    },
    'centos8': {
      node {
        sh 'setup_centreon_build.sh'
        sh "./centreon-build/jobs/widgets/${serie}/widget-package.sh centos8"
      }
    }
    if ((currentBuild.result ?: 'SUCCESS') != 'SUCCESS') {
      error('Package stage failure.');
    }
  }

  if ((env.BUILD == 'RELEASE') || (env.BUILD == 'REFERENCE')) {
    stage('Delivery') {
      node {
        sh 'setup_centreon_build.sh'
        sh "./centreon-build/jobs/widgets/${serie}/widget-delivery.sh"
      }
      if ((currentBuild.result ?: 'SUCCESS') != 'SUCCESS') {
        error('Delivery stage failure.');
      }
    }
  }
} catch(e) {
  if ((env.BUILD == 'RELEASE') || (env.BUILD == 'REFERENCE')) {
    slackSend channel: "#monitoring-metrology",
        color: "#F30031",
        message: "*FAILURE*: `CENTREON WIDGET TACTICAL OVERVIEW` <${env.BUILD_URL}|build #${env.BUILD_NUMBER}> on branch ${env.BRANCH_NAME}\n" +
            "*COMMIT*: <https://github.com/centreon/centreon-widget-tactical-overview/commit/${source.COMMIT}|here> by ${source.COMMITTER}\n" +
            "*INFO*: ${e}"
  }

  currentBuild.result = 'FAILURE'
}
