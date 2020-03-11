#!/usr/bin/env groovy

import java.util.Date
import groovy.json.*


def repoName = 'marketing-console-campaign'
def projectName = 'global'
def deploymentName = 'marketing-console-campaign'

def isMaster = env.BRANCH_NAME == 'master'
def isStaging = env.BRANCH_NAME == 'staging'
def start = new Date()
def err = null

String jobInfoShort = "${env.JOB_NAME} ${env.BUILD_DISPLAY_NAME}"
String jobInfo = "${env.JOB_NAME} ${env.BUILD_DISPLAY_NAME} \n${env.BUILD_URL}"
String buildStatus
String timeSpent

currentBuild.result = "SUCCESS"

try {
    node {
        deleteDir()

        stage('initializing'){
            slackSend (color: 'good', message: "Initializing build process for `${repoName}` ...")
        }

        stage ('Checkout') {
            checkout scm
        }
               

      stage ('Push Docker to AWS ECR') {
            def tag = isMaster ? "latest" : "staging"
            sh "\$(aws ecr get-login --no-include-email --region ${AWS_ECR_REGION})"
            sh "docker build -t ${AWS_ECR_ACCOUNT}/${projectName}/${repoName}:${tag} ."
            pushImage(repoName, projectName, tag)
            slackSend (color: 'good', message: "docker image on `${env.BRANCH_NAME}` branch in `${repoName}` pushed to *_AWS ECR_*")
        }

      if(isMaster || isStaging){
            def tag = isMaster ? "latest" : "staging"
            stage ('Deploy to Kubernetes') {
                deploy(deploymentName, tag)
                slackSend (color: 'good', message: ":fire: Nice work! `${repoName}` deployed to *_Kubernetes_*")
            }      
      
      } 
     stage('Clean up'){
                 def tag = isMaster ? "latest" : "staging"
                sh "docker rmi ${AWS_ECR_ACCOUNT}/${projectName}/${repoName}:${tag}"
         }
    }
} catch (caughtError) {
    err = caughtError
    currentBuild.result = "FAILURE"

} finally {
     timeSpent = "\nTime spent: ${timeDiff(start)}"

    if (err) {
        slackSend (color: 'danger', message: ":disappointed: _Build failed_: ${jobInfo} ${timeSpent} ${err}")
    } else {
        if (currentBuild.previousBuild == null) {
            buildStatus = '_First time build_'
        } else if (currentBuild.previousBuild.result == 'SUCCESS') {
            buildStatus = '_Build complete_'
        } else {
            buildStatus = '_Back to normal_'
        }
        slackSend (color: 'good', message: "${jobInfo}: ${timeSpent}")
    }
}

def deploy(deploymentName, tag){
    def namespace = tag == "latest" ? K8S_PRODUCTION_PATH : K8S_STAGING_PATH
    try{
        sh "kubectl apply  -f kubernetes/"
    }
    catch (err) {
        slackSend (color: 'danger', message: ":disappointed: _Build failed_: ${jobInfo} ${err}")
    }   
}

def pushImage(repoName, projectName, tag){
    try{
        sh "docker push ${AWS_ECR_ACCOUNT}/${projectName}/${repoName}:${tag}"
    }catch(e){
        sh "aws ecr create-repository --repository-name ${projectName}/${repoName} --region ${AWS_ECR_REGION}"
         sh "aws ecr set-repository-policy --repository-name ${projectName}/${repoName} --policy-text file://policy.json --region ${AWS_ECR_REGION}"
        sh "docker push ${AWS_ECR_ACCOUNT}/${projectName}/${repoName}:${tag}"
    }
}

def timeDiff(st) {
    def delta = (new Date()).getTime() - st.getTime()
    def seconds = delta.intdiv(1000) % 60
    def minutes = delta.intdiv(60 * 1000) % 60
    return "${minutes} min ${seconds} sec"
}