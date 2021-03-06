pipeline {

  agent { node { label 'ios' } }

  options {
    buildDiscarder(logRotator(numToKeepStr: '3'))
  }

  environment {
    GIT_COMMITTER_NAME = 'nimbl3-dev'
    GIT_COMMITTER_EMAIL = 'dev@nimbl3.com'
    IS_JENKINS = 'true'
    COMMIT_MESSAGE = getCommitMessage()
  }

  stages {

    stage('Debug environment') {
      steps {
        sh 'whoami'
        sh 'env'
      }
    }

    stage('Setup CI') {
      when { not { expression { shouldSkipCI() } } } 
      steps {
        sh 'bundle install --path .bundle'
        sh 'bundle exec fastlane run setup_jenkins'
        sh 'bundle exec fastlane ci_clean_up'
        sh 'bundle exec fastlane setup_cocoapods'
      }
    }

    stage('Linting') {
      when {
        not { expression { shouldSkipCI() } }
        branch 'PR-*'
      }

      environment {
        BITBUCKET_CREDENTIALS = credentials('__GIT_BITBUCKET_ACCOUNT_CREDENTIAL_JENKINS_CREDENTIAL_ID__')      
        PRONTO_BITBUCKET_USERNAME = "${BITBUCKET_CREDENTIALS_USR}"
        PRONTO_BITBUCKET_PASSWORD = "${BITBUCKET_CREDENTIALS_PSW}"
        PRONTO_PULL_REQUEST_ID = "${env.CHANGE_ID}"
        PRONTO_SWIFTLINT_PATH = "${env.WORKSPACE}/Pods/SwiftLint/swiftlint"
      }
      steps {
        sh "bundle exec pronto run -f bitbucket_pr -c upstream/${env.CHANGE_TARGET}"
      }
    }
    
    stage('Build for testing') {
      when { not { expression { shouldSkipCI() } } }
      steps {
        sh 'bundle exec fastlane ci_build_for_testing'
      }
    }

    stage('Unit tests') {
      when { not { expression { shouldSkipCI() } } }
      steps {
        sh 'bundle exec fastlane ci_unit_tests'
      }
    }

    stage('Instrumented & UI tests') {
      when { not { expression { shouldSkipCI() } } }
      steps {
        script {
          if (currentBuild.result == null || currentBuild.result == 'SUCCESS') {
            //todo:- later add a flag to run only for critical for wip branch
            sh 'bundle exec fastlane ci_ui_tests'
          }
        }
      }
    }

    stage('Build for deploy') {
      when {
        not { expression { shouldSkipCI() } }
        not { expression { shouldSkipDeploy() } }
        anyOf {
          branch 'develop'
          branch 'release/*'
          branch 'master'
        }
      }
      environment {
        MATCH_PASSWORD = credentials("__FASTLANE_MATCH_PASSWORD_JENKINS_CREDENTIAL_ID__")
        ITUNES_CONNECT_CREDENTIALS = credentials("__ITUNE_CONNECT_JENKINS_CREDENTIAL_ID__")
        FASTLANE_USER = "${ITUNES_CONNECT_CREDENTIALS_USR}"
        FASTLANE_PASSWORD = "${ITUNES_CONNECT_CREDENTIALS_PSW}"
        FASTLANE_DONT_STORE_PASSWORD = "1"        
      }
      steps {
        script {
          withCredentials(bindings: [sshUserPrivateKey(credentialsId: '__GIT_SSH_JENKINS_CREDENTIAL_ID__', keyFileVariable: 'SSH_PRIVATE_KEY')]) {
            sh '''
              eval $(ssh-agent) && ssh-add $SSH_PRIVATE_KEY
              ssh git@bitbucket.org
              bundle exec fastlane ci_code_signing_for_deploy
            '''
          }
          if (env.BRANCH_NAME ==~ /develop/) {
            sh 'bundle exec fastlane ci_build_for_internal_beta'
          } else if (env.BRANCH_NAME =~ /release\//) {
            sh 'bundle exec fastlane ci_build_for_beta'
          } else if (env.BRANCH_NAME ==~ /master/) {
            //todo:- change to release_appstore once the app is published
          }
        }
      }
    }

    stage('Deploy') {
      when {
        not { expression { shouldSkipCI() } }
        not { expression { shouldSkipDeploy() } }
        anyOf {
          branch 'develop'
          branch 'release/*'
          branch 'master'
        }
      }
      steps {
        script {
          if (currentBuild.result == null || currentBuild.result == 'SUCCESS') {
            if (env.BRANCH_NAME ==~ /develop/) {
              sh 'bundle exec fastlane ci_upload_to_crashlytics is_internal:true'
              sh 'bundle exec fastlane ci_upload_dSYM'
            } else if (env.BRANCH_NAME =~ /release\//) {
              sh 'bundle exec fastlane ci_upload_to_crashlytics is_internal:false'
              sh 'bundle exec fastlane ci_upload_dSYM'
              withCredentials(bindings: [sshUserPrivateKey(credentialsId: '__GIT_SSH_JENKINS_CREDENTIAL_ID__', keyFileVariable: 'SSH_PRIVATE_KEY')]) {
                sh '''
                  eval $(ssh-agent) && ssh-add $SSH_PRIVATE_KEY
                  ssh git@bitbucket.org
                  bundle exec fastlane ci_commit_bump_build_and_push_changes
                '''
              }
              sh 'bundle exec fastlane ci_create_release_pull_request'
            } else if (env.BRANCH_NAME ==~ /master/) {
              //todo:- change to release_appstore once the app is published
            }
            sh 'bundle exec fastlane remove_keychain'
          } else {
            echo 'Found an issue when building. Deployment failed'
          }
        }
      }
    }

    stage("Clean git status") {
      when {
        not { expression { shouldSkipCI() } }
        not { expression { shouldSkipDeploy() } }
      }
      steps {
        sh 'bundle exec fastlane ci_clean_git_repository'
      }
    }
  }

  post {
    always {
      archiveArtifacts(allowEmptyArchive: true, artifacts: 'fastlane/test_output/*.zip', fingerprint: true)
      archiveArtifacts(allowEmptyArchive: true, artifacts: 'fastlane/test_output/report.html', fingerprint: true)
      archiveArtifacts(allowEmptyArchive: true, artifacts: 'build/*.ipa', fingerprint: true)
      archiveArtifacts(allowEmptyArchive: true, artifacts: 'build/*.dSYM.zip', fingerprint: true)
    }
  }
}