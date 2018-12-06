/**
 * The openstack-ardana-update-and-test Jenkins Pipeline
 *
 * This job updates a pre-deployed CLM cloud and run tests.
 */

pipeline {

  options {
    // skip the default checkout, because we want to use a custom path
    skipDefaultCheckout()
    timestamps()
  }

  agent {
    node {
      label reuse_node ? reuse_node : "cloud-ardana-ci"
      customWorkspace "${JOB_NAME}-${BUILD_NUMBER}"
    }
  }

  stages {
    stage('Setup workspace') {
      steps {
        script {
          // Set this variable to be used by upstream builds
          env.blue_ocean_buildurl = env.RUN_DISPLAY_URL
          if (ardana_env == '') {
            error("Empty 'ardana_env' parameter value.")
          }
          // Parameters of the type 'extended-choice' are set to null when the job
          // is automatically triggered and its default value is ''. So, we need to set
          // it to '' to be able to pass it as a parameter to downstream jobs.
          if (env.tempest_filter_list == null) {
            env.tempest_filter_list = ''
          }
          if (env.qa_test_list == null) {
            env.qa_test_list = ''
          }
          currentBuild.displayName = "#${BUILD_NUMBER}: ${ardana_env}"
          // Use a shared workspace folder for all jobs running on the same
          // target 'ardana_env' cloud environment
          env.SHARED_WORKSPACE = sh (
            returnStdout: true,
            script: 'echo "$(dirname $WORKSPACE)/shared/${ardana_env}"'
          ).trim()
          if (reuse_node == '') {
            sh('''
              rm -rf $SHARED_WORKSPACE
              mkdir -p $SHARED_WORKSPACE

              # archiveArtifacts and junit don't support absolute paths, so we have to to this instead
              ln -s ${SHARED_WORKSPACE}/.artifacts ${WORKSPACE}

              cd $SHARED_WORKSPACE
              git clone $git_automation_repo --branch $git_automation_branch automation-git
              source automation-git/scripts/jenkins/ardana/jenkins-helper.sh
              ansible_playbook load-job-params.yml
              ansible_playbook setup-ssh-access.yml -e @input.yml
            ''')
          }
          env.DEPLOYER_IP = sh (
            returnStdout: true,
            script: '''
              grep -oP "^${ardana_env}\\s+ansible_host=\\K[0-9\\.]+" \\
                $SHARED_WORKSPACE/automation-git/scripts/jenkins/ardana/ansible/inventory
            '''
          ).trim()
          currentBuild.displayName = "#${BUILD_NUMBER}: ${ardana_env} (${DEPLOYER_IP})"
        }
      }
    }

    stage('Update ardana') {
      steps {
        script {
          def slaveJob = build job: 'openstack-ardana-update', parameters: [
            string(name: 'ardana_env', value: "$ardana_env"),
            string(name: 'update_to_cloudsource', value: "$update_to_cloudsource"),
            string(name: 'updates_test_enabled', value: "$updates_test_enabled"),
            string(name: 'maint_updates', value: "$maint_updates"),
            string(name: 'rc_notify', value: "$rc_notify"),
            string(name: 'git_automation_repo', value: "$git_automation_repo"),
            string(name: 'git_automation_branch', value: "$git_automation_branch"),
            string(name: 'reuse_node', value: "${NODE_NAME}"),
            string(name: 'os_cloud', value: "$os_cloud")
          ], propagate: true, wait: true
        }
      }
    }

    stage ('Tempest/QA') {
      when {
        expression { tempest_filter_list != '' || qa_test_list != '' }
      }
      steps {
        catchError {
          script {
            def slaveJob = build job: 'openstack-ardana-tests', parameters: [
              string(name: 'ardana_env', value: "$ardana_env"),
              string(name: 'tempest_filter_list', value: "$tempest_filter_list"),
              string(name: 'qa_test_list', value: "$qa_test_list"),
              string(name: 'rc_notify', value: "$rc_notify"),
              string(name: 'git_automation_repo', value: "$git_automation_repo"),
              string(name: 'git_automation_branch', value: "$git_automation_branch"),
              string(name: 'reuse_node', value: "${NODE_NAME}"),
              string(name: 'os_cloud', value: "$os_cloud")
            ], propagate: true, wait: true
          }
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: ".artifacts/**/*", allowEmptyArchive: true
      junit testResults: ".artifacts/*.xml", allowEmptyResults: true
    }
    cleanup {
      cleanWs()
    }
  }
}
