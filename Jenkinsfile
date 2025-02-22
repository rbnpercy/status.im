pipeline {
  agent { label 'linux' }

  options {
    disableConcurrentBuilds()
    /* manage how many builds we keep */
    buildDiscarder(logRotator(
      numToKeepStr: '20',
      daysToKeepStr: '30',
    ))
  }

  environment {
    GIT_USER = 'status-im-auto'
    GIT_MAIL = 'auto@status.im'
  }

  stages {
    stage('Git Prep') {
      steps {
        sh "git config user.name ${GIT_USER}"
        sh "git config user.email ${GIT_MAIL}"
        /* necessary to have access to the theme partials */
        sshagent(credentials: ['status-im-auto-ssh']) {
          sh 'git submodule update --init --recursive'
        }
      }
    }

    stage('Install Deps') {
      steps {
        sh 'npm install'
        sh 'npm update'
      }
    }

    stage('Index') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'search.status.im-auth',
          usernameVariable: 'HEXO_ES_USER',
          passwordVariable: 'HEXO_ES_PASS'
        )]) {
          sh 'npm run index'
        }
      }
    }

    stage('Build') {
      steps {
        sh 'npm run clean'
        sh 'npm run build'
      }
    }

    stage('Robot') {
      when { expression { !GIT_BRANCH.endsWith('master') } }
      steps { script {
        sh 'echo "Disallow: /" >> public/robots.txt'
      } }
    }

    stage('Publish Prod') {
      when { expression { GIT_BRANCH.endsWith('master') } }
      steps { script {
        sshagent(credentials: ['status-im-auto-ssh']) {
          sh 'npm run deploy'
        }
      } }
    }

    stage('Publish Devel') {
      when { expression { !GIT_BRANCH.endsWith('master') } }
      steps { script {
        sshagent(credentials: ['jenkins-ssh']) {
          sh '''
            rsync -e 'ssh -o StrictHostKeyChecking=no' -r --delete public/. \
            jenkins@node-01.do-ams3.proxy.misc.statusim.net:/var/www/dev/
          '''
        }
      } }
    }
  }
}
