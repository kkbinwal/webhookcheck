#!/usr/bin/env groovy
import groovy.json.JsonSlurper


customImage = ""
cycleStage = ""
//EMAIL VARIABLES
EMAIL_INFO = "No information available"

// GLOBAL VARIABLES
HAS_TIMING_ISSUE = false
TIMING_MESSAGE = ""
manifest = ""
manifestfile = ""
PROJECT_LOC = ""
ARTIFACTORY_BUILD_NAME = 'Molex IAS IOT Services'
JIRA_PROJECT_KEY = 'IASI'
DOCKER_REPO = 'iss-docker-dev'
EXCLUDE_PROJECT = "molex.iot.prov.fidoiot"
PROJECT_URL = 'https://bitbucket.atlassian.molexcloud.com/scm/iasi/'


def isValidPR(hookPayload) {
    def payload = readJSON file: hookPayload
    def eventKey = payload.eventKey
    if(eventKey != null) {
        if(eventKey.startsWith("pr")) {
            def srcBranch = payload.pullRequest.fromRef.id.split('/')[-1]
            def targetBranch = payload.pullRequest.toRef.id.split('/')[-1]
            def error_msg = "PR Error: Either source branch or  target branch should be master"
            if(srcBranch == null || targetBranch == null || (!srcBranch.equals("master") && !targetBranch.equals("master"))){
                print("${error_msg}")
                return false
            }
            return false
        }
    }
    return true
}


pipeline {
  agent {
    label 'agent1'
  }
  stages {
    stage('Configure Settings') {
     
      steps {
        script {
          echo "Configuring Settings"

             

          try {

            //def inputFile = new File('sample_webhook.json')
            if(isValidPR('sample_webhook.json')){
              print("valid PR")
            }  
            else{
              print("invalid PR")
            }          
          } catch(Exception e) {
            echo "No payload present, using default configuration."
              print("${e.message}")

          }

        }
      }
    }

  }

}
