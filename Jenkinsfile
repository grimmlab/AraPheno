#!/usr/bin/env groovy


/**
* build / test pipeline for AraPheno
*/

pipeline {

    agent any

    stages {

        // global checkout needed for Docker setup
        stage('checkout') {
            steps {
                checkout scm
            }
        }

        // prepare the dockers: testing env
        stage('test') {
            steps {
                script {

                    def app_img = docker.build("arapheno")

                    // link app container to db container, expose platinum test config path
                    app_img.inside() {
                        // run the behave tests
                        echo "running tests"
                        sh returnStatus: true, script: "cd /srv/web/ && py.test -ra -p no:cacheprovider --junitxml ${env.WORKSPACE}/TESTS-arapheno.xml"
                    }
                     // collect unit test results
                    junit allowEmptyResults: true, testResults: "TESTS-arapheno.xml"
                }
            }
        }

        // not _really_ deploy, but push to local registry
        stage('deploy') {
            when{
                branch 'master'
            }
            steps {
                script {
                    sh 'sh write_version.sh'
                    //FIXME leaking local infra details
                    docker.withRegistry('https://docker.artifactory.imp.ac.at', 'jenkins_artifactory_creds') {

                        // push DB docker img to registry
                        def server_img = docker.build("docker.artifactory.imp.ac.at/the1001genomes/arapheno:${env.BUILD_ID}")

                        server_img.push('latest')
                        sshagent(['1001genome_deploy_ssh_key']) {
                            env.DEPLOY_HOST = 'arapheno.sci.gmi.oeaw.ac.at'
                            sh '''
                                scp -o StrictHostKeyChecking=no docker-compose.yml root@$DEPLOY_HOST:/root/
                                ssh -o StrictHostKeyChecking=no root@$DEPLOY_HOST "cd /root && docker-compose pull && docker-compose up -d"
                            '''
                        }
                    }
                }
            }
        }
    // in the future: conditional deploy?
    } //stages

    // post block is executed after all the stages/steps in the pipeline
    post {
        // always {
        //     // notify build results, see https://jenkins.io/blog/2016/07/18/pipline-notifications/
        //     // notifyBuild(currentBuild.result)

        // }
        changed {
            echo "build changed"
        }
        aborted {
            echo "build aborted"
        }
        failure {
            echo "build failed"
        }
        success {
            echo "build is success"
        }
        unstable {
            echo "build is unstable"
        }
    }
}

// send global slack notification, but fail silently if this is not possible, i.e. slack integration is not installed
def notifyBuild(String buildStatus) {
  // build status of null means successful
  buildStatus = buildStatus ?: 'SUCCESS'

  if (buildStatus == 'STARTED' || buildStatus == 'CHANGED' || buildStatus == 'UNSTABLE') {
    color = 'YELLOW'
    colorCode = '#DDDD00'
  }
  else if (buildStatus == 'SUCCESS') {
    color = 'GREEN'
    colorCode = '#00DD00'
  }
  else {
    // FAILURE or UNSTABLE
    color = 'RED'
    colorCode = '#DD0000'
  }

  def message = "${buildStatus}: Job <${env.BUILD_URL}|${env.JOB_NAME} #${env.BUILD_NUMBER}>"

  /* we could add blame to the slack message ;) but it's visible on the build details anyway
  if (buildStatus == 'FAILURE') {
      def changes_by = sh 'git --no-pager log -1 --format=%an'
      message = "${buildStatus}: Job <${env.BUILD_URL}|${env.JOB_NAME} #${env.BUILD_NUMBER}> caused by ${changes_by}"
  }
  */

  // send gracefully
  try {
      slackSend (color: colorCode, message: "${message}")
  }
  catch (e) {
     echo "failed to send notification: ${e}"
  }
}