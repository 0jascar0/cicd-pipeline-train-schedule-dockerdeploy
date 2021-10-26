pipeline {
    agent any
      tools {nodejs "node"}
 
    environment {
        //be sure to replace "jascar" with your own Docker Hub username
        DOCKER_IMAGE_NAME = "jascar/trainschedule"
        CHKP_CLOUDGUARD_CREDS = credentials("CloudGuard_Credentials")
    }
      
    stages {
     stage('Check') {
  when {
    anyOf {
      changeset "src/**/*.ts"
      changeset "package.json"
    }
  }
  steps {
      sh 'npm install'
      sh 'npm run Build'
  }
}        

               
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build(DOCKER_IMAGE_NAME)
//                    app.inside {
//                        sh 'echo $(curl localhost:8081)'
//                    }
                }
            }
        }
        stage('CloudGuard_Shiftleft_Code_Scan') {
            environment {
                CHKP_CLOUDGUARD_CREDS = credentials("${CHKP_CLOUDGUARD_CREDS}")
                        }
            agent {
                docker { 
                    image (DOCKER_IMAGE_NAME)
                    args '-v /tmp/:/tmp/'
                  }
                        }
            steps {
                dir('code-dir') {
                   credentialsId: (github-key-new)
                    url: "https://github.com/0jascar0/cicd-pipeline-train-schedule-dockerdeploy.git"
                }
                sh '''
                    export CHKP_CLOUDGUARD_ID=$CHKP_CLOUDGUARD_CREDS_USR
                    export CHKP_CLOUDGUARD_SECRET=$CHKP_CLOUDGUARD_CREDS_PSW
                    shiftleft code-scan -s code-dir -r {rulesetId} -e {environmentId}
                '''
                    }
    }
        stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerhub_id') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('ShiftLeft Code Scan') {   
      steps {
                dir('iac-code') {
                    git branch: '{banch}',
                    credentialsId: '{github-key}',
                     url: "https://github.com/0jascar0/cicd-pipeline-train-schedule-dockerdeploy.git"
                }
                sh '''
                    export CHKP_CLOUDGUARD_ID=$CHKP_CLOUDGUARD_CREDS_USR
                    export CHKP_CLOUDGUARD_SECRET=$CHKP_CLOUDGUARD_CREDS_PSW
                    shiftleft iac-assessment -i terraform -p iac-code/terraform-template -r {rulesetId} -e {environmentId}
                '''
            }
        }
    
    stage('DeployToProduction') {
            when {
                branch 'master'
            }
            steps {
                input 'Deploy to Production?'
                milestone(1)
                withCredentials([usernamePassword(credentialsId: 'webserver_login', usernameVariable: 'USERNAME', passwordVariable: 'USERPASS')]) {
                    script {
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker pull willbla/train-schedule:${env.BUILD_NUMBER}\""
                        try {
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker stop train-schedule\""
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker rm train-schedule\""
                        } catch (err) {
                            echo: 'caught error: $err'
                        }
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker run --restart always --name train-schedule -p 8080:8080 -d willbla/train-schedule:${env.BUILD_NUMBER}\""
                    }
                }
            }
        }
    }
}
