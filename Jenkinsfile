#!/usr/bin/env groovy



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
    def payload = readJSON text: hookPayload

    def eventKey = payload.eventKey

    if(eventKey != null) {
        if(eventKey.startsWith("pr")) {
            def fromMaster = eventKey.pullRequest.fromRef.id.split('/')[-1]
            def toMaster = eventKey.pullRequest.toRef.id.split('/')[-1]
            if(fromMaster == null || toMaster == null)return false
            if(fromMaster.equals("master") || toMaster.equals("master")) {
                return true
            } 
            print("PR is neither from master branch not to master branch")
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

            def inputFile = new File('.//sample_webhook.json')
            if(isValidPR(inputFile)){
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
