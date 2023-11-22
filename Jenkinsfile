pipeline {
  options {
    timeout(time: 60, unit: 'MINUTES')
    ansiColor('xterm')
    disableConcurrentBuilds(abortPrevious: true)
    buildDiscarder logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '5', numToKeepStr: '5')
  }

  agent {
    label 'node'
  }

  environment {
    NODE_ENV = 'production'
    TZ = "UTC"
    NETLIFY = "true"
    // Amount of available vCPUs, to avoid OOM - https://www.gatsbyjs.com/docs/how-to/performance/resolving-out-of-memory-issues/#try-reducing-the-number-of-cores
    // https://github.com/jenkins-infra/jenkins-infra/tree/production/hieradata/clients/controller.ci.jenkins.io.yaml#L327
    GATSBY_CPU_COUNT = "4"
  }

  stages {
    stage('Check for typos') {
      steps {
        sh '''
          curl -qsL https://github.com/crate-ci/typos/releases/download/v1.5.0/typos-v1.5.0-x86_64-unknown-linux-musl.tar.gz | tar xvzf - ./typos
          curl -qsL https://github.com/halkeye/typos-json-to-checkstyle/releases/download/v0.1.1/typos-checkstyle-v0.1.1-x86_64 > typos-checkstyle && chmod 0755 typos-checkstyle
          ./typos --format json | ./typos-checkstyle - > checkstyle.xml || true
        '''
      }
      post {
        always {
          recordIssues(tools: [checkStyle(id: 'typos', name: 'Typos', pattern: 'checkstyle.xml')])
        }
      }
    }

    stage('Install Dependencies') {
      environment {
        NODE_ENV = 'development'
      }
      steps {
        sh 'asdf install'
        sh 'npm install'
      }
    }

    stage('Build PR') {
      when { changeRequest() }
      environment {
        NODE_ENV = 'development'
      }
      steps {
        sh 'npm run build'
      }
    }

    stage('Deploy PR to preview site') {
      when {
        allOf{
          changeRequest target: 'main'
          // Only deploy to production from infra.ci.jenkins.io
          expression { infra.isInfra() }
        }
      }
      environment {
        NETLIFY_AUTH_TOKEN = credentials('netlify-auth-token')
      }
      steps {
        sh 'netlify-deploy --draft=true --siteName "contributor-spotlight" --title "Preview deploy for ${CHANGE_ID}" --alias "deploy-preview-${CHANGE_ID}" -d ./public'
      }
      post {
        success {
          recordDeployment('jenkins-infra', 'contributor-spotlight', pullRequest.head, 'success', "https://deploy-preview-${CHANGE_ID}--contributor-spotlight.netlify.app")
        }
        failure {
          recordDeployment('jenkins-infra', 'contributor-spotlight', pullRequest.head, 'failure', "https://deploy-preview-${CHANGE_ID}--contributor-spotlight.netlify.app")
        }
      }
    }
  }
}
