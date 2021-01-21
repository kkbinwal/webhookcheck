#!/usr/bin/env groovy
@Library("jenkinsLibrary") _


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


def configureSonarQube(){
  def branch = settings.PROJECT_DETAILS['branch'].split('/')[-1]
  def SONAR_OPTIONS = []
  if (settings.PROJECT_DETAILS['eventType'].equals('pull')) {
      sh 'echo this is a pull request'
      def WEBHOOK_SETTINGS =  readJSON text: webhookPayload

     SONAR_OPTIONS ="-Dsonar.scm.forceReloadAll=true \
                      -Dsonar.scm.revision=${WEBHOOK_SETTINGS.pullRequest.fromRef.latestCommit} \
                      -Dsonar.pullrequest.key=${WEBHOOK_SETTINGS.pullRequest.id} \
                      -Dsonar.pullrequest.bitbucketserver.headSha=${WEBHOOK_SETTINGS.pullRequest.fromRef.latestCommit} \
                      -Dsonar.pullrequest.bitbucketserver.triggerCommit=${WEBHOOK_SETTINGS.pullRequest.fromRef.latestCommit} \
                      -Dsonar.analysis.prNumber=${WEBHOOK_SETTINGS.pullRequest.id} \
                      -Dsonar.analysis.sha1=${WEBHOOK_SETTINGS.pullRequest.fromRef.latestCommit} \
                      -Dsonar.pullrequest.branch=${WEBHOOK_SETTINGS.pullRequest.fromRef.id.split('/')[-1]} \
                      -Dsonar.pullrequest.base=${WEBHOOK_SETTINGS.pullRequest.toRef.id.split('/')[-1]}"

  } else {
      SONAR_OPTIONS="-Dsonar.branch.name=${branch}"
  }
  return SONAR_OPTIONS
}
def runXRay() {
    STAGE_FAIL = env.STAGE_NAME
    checkout([
        $class: 'GitSCM',
        branches: [[name: 'master' ]],
        doGenerateSubmoduleConfigurations: false,
        extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'jfrog']],
        submoduleCfg: [],
        userRemoteConfigs: [[
            credentialsId: "${settings.GIT_CREDS}",
            url: 'https://bitbucket.atlassian.molexcloud.com/scm/id/jfrog-code-insights.git'
        ]]
    ]).GIT_COMMIT
    xrayScan(
        serverId: "${settings.ART_SERVER}",
        buildName: ARTIFACTORY_BUILD_NAME,
        buildNumber: "${BUILD_NUMBER}",
        failBuild: false
    )
    print ("settings.PROJECT_DETAILS['eventType'] is: ${settings.PROJECT_DETAILS['eventType']}")
    if (settings.PROJECT_DETAILS['eventType'].equals('pull')) {
        def WEBHOOK_SETTINGS =  readJSON text: webhookPayload
        sh "echo ${BUILD_NUMBER}"
        sh 'cat jfrog/build.env > tempFile'
        sh "sed -e \"s,PRJ-KEY,${JIRA_PROJECT_KEY},g\" \
        -e \"s,REPO-NAME,${REPO_NAME},g\" \
        -e \"s,CI-BUILD-NAME-URL,${ARTIFACTORY_BUILD_NAME},g\" \
        -e \"s,CI-BUILD-NAME,${ARTIFACTORY_BUILD_NAME},g\" \
        -e \"s,CI-BUILD-NUM,${BUILD_NUMBER},g\" \
        -e \"s,GIT-COMMIT,${WEBHOOK_SETTINGS.pullRequest.fromRef.latestCommit},g\" tempFile > jfrog/build.env"
        sh 'cd jfrog ; ./build.sh'
    }
    else{
      print ("Not triggering xray as eventType is not pull")
    }
}

def deployToArtifactory() {
    STAGE_FAIL = env.STAGE_NAME
    def branch = settings.PROJECT_DETAILS['branch'].split('/')[-1]
    ARTIFACTORY_DOCKER_URL = "${DOCKER_REPO}.${settings.ARTIFACTORY_DNS}"
    IMAGE_NAME = "${ARTIFACTORY_DOCKER_URL}/${IMAGE_ARTIFACT_ID}:${IMAGE_VERSION}.${env.BUILD_ID}"
    if(branch.equals("master")){
      IMAGE_NAME = "${ARTIFACTORY_DOCKER_URL}/${IMAGE_ARTIFACT_ID}:${branch}"
    }
    withCredentials(
    [usernamePassword(
        credentialsId: 'art-service',
        passwordVariable: 'ART_PASS',
        usernameVariable: 'ART_USER')]) {
            sh "docker login -u ${ART_USER} -p ${ART_PASS} docker-hub.artifactory.molexcloud.com"
            sh 'pwd && ls -lha'
            sh "docker build -t ${IMAGE_NAME} ."
    }


    rtBuildInfo(
        buildName: ARTIFACTORY_BUILD_NAME,
        buildNumber: "${BUILD_NUMBER}",
        captureEnv: true
    )

    rtDockerPush(
        serverId: 'art-qa',
        image: IMAGE_NAME,
        targetRepo: DOCKER_REPO,
        buildName: ARTIFACTORY_BUILD_NAME,
        buildNumber: "${BUILD_NUMBER}"
    )

    rtPublishBuildInfo (
        serverId: "${settings.ART_SERVER}",
        buildName: ARTIFACTORY_BUILD_NAME,
        buildNumber: "${BUILD_NUMBER}",
    )

    def targetRepo = 'iss-docker-prod'

    if (DOCKER_REPO.equals('iss-docker-qa')) {
        rtAddInteractivePromotion (
            serverId: "${settings.ART_SERVER}",
            buildName: ARTIFACTORY_BUILD_NAME,
            buildNumber: "${BUILD_NUMBER}",
            sourceRepo: DOCKER_REPO,
            targetRepo: targetRepo,
            status: 'Promote to Production',
            copy: true
        )
    }
}


pipeline {
  agent {
    label 'aws-ubuntu01'
  }
  stages {
    stage('Configure Settings') {
     
      steps {
        script {
          echo "Configuring Settings"

          def REPOS_CONF = readYaml file:"repos.yml"
          def REPOS_LIST = REPOS_CONF.keySet() as List
          REPO_NAME = targetProject

          echo "${REPOS_CONF}"
			
          project_repos = [:]
          REPOS_CONF.each { key, value ->
            if(key.equals(targetProject))
            {
              repo_info = [:]
              repo_info['branch'] = "refs/heads/${TARGET_BRANCH}"
              repo_info['hash'] = ''
              repo_info['url'] = value.url
              repo_info['ssh'] = value.ssh
              project_repos[key] = repo_info
            }
          }
          print("Config: ${settings.PROJECT_REPOS}")

          settings.initDefaults()
          settings.PROJECT_DETAILS['name'] = REPOS_LIST[0]
           

          try {
            echo webhookPayload
            settings.configureProjectSettings(webhookPayload)

            if(settings.PROJECT_DETAILS.eventType.equals("pull_merge")) {
              buildType = "pre-production"
            }
            targetProject = settings.PROJECT_DETAILS['name']
            REPO_NAME = targetProject
            
          } catch(Exception e) {
            log.print_debug("${e.message}")
            echo "No payload present, using default configuration."
            settings.PROJECT_REPOS = project_repos

          }
          print("Config: ${settings.PROJECT_REPOS}")
          print("targetProject: ${targetProject}")
          settings.PROJECT_DETAILS['branch'] = settings.PROJECT_REPOS[targetProject]['branch']
          IMAGE_ARTIFACT_ID = targetProject
        }
      }
    }
    stage ('Build'){
            agent {
              dockerfile {
                filename 'dockerfile'
                label 'aws-ubuntu01'
                registryUrl 'https://docker-hub.artifactory.molexcloud.com'
                registryCredentialsId 'art-service'
                args '-u root:root --init'
              }
            }
      stages{
        stage('Maven Build'){
         
        stages {  
          stage('Checkout') {
            steps {
              script {
               print("cloning repos")
               parallel utils.cloneRepos(settings.PROJECT_REPOS, "https")
               print("setting project details for targetProject:"+targetProject)
               settings.PROJECT_DETAILS['hash'] = settings.PROJECT_REPOS["${targetProject}"]['hash']
               sh"pwd"
              }
            }
          }
          stage('Test') {
            steps {
              script {
                print("build")
                sh"ls -la"
                sh"pwd"
                

                  dir("${targetProject}") {
                    withCredentials([usernamePassword(credentialsId: 'art-service',usernameVariable: 'username',passwordVariable: 'password')]) {
                      print("Running mvn test ")
                      
                      sh "mvn -Drepo.login=${username}  -Drepo.pwd=${password} test "

                    }  
                  }              
              }
            }
          }
          stage('Build') {
            steps {
              script {
                print("build")
                sh"ls -la"
                sh"pwd"
                  dir("${targetProject}") {
                    withCredentials([usernamePassword(credentialsId: 'art-service',usernameVariable: 'username',passwordVariable: 'password')]) {
                      print("Running mvn clean deploy ")
                      
                      sh 'mvn "-Drepo.login=${username}"  "-Drepo.pwd=${password}" clean deploy '
                      if(!(targetProject.equals("molex.iot.prov.fidoiot"))){
                      	pom = readMavenPom file: 'pom.xml'
                      	IMAGE_VERSION = pom.parent.version
                      	/*IMAGE_ARTIFACT_ID = pom.artifactId*/
                      	stash name:"dockerfile", includes:"dockerfile"
                      	stash name:"jarfile", includes:"target/output-jar-with-dependencies.jar"
                      }
                    }  
                  }              
                
                
              }
            }
          }

          stage('SonarAnalysis') {
          
            steps {
              script {
                dir("${targetProject}") {
                  withSonarQubeEnv('SonarqubeCorp'){
                    withCredentials([usernamePassword(credentialsId: 'art-service',usernameVariable: 'username',passwordVariable: 'password')]) {
                      print("Running mvn sonar analysis with sonar host url: ${SONAR_HOST_URL}")
                      SONAR_OPTIONS = configureSonarQube()
                      sh "mvn  -Dsonar.url=${SONAR_HOST_URL} -Drepo.login=${username}   -Drepo.pwd=${password} -Dsonar.projectKey=${targetProject} -Dsonar.phase=verify   ${SONAR_OPTIONS} sonar:sonar"
                    }
                  }
                }
              }
            }
          }



        }    
          post { //Cleaning WS
            always {
              echo "Cleaning files"
              //cleanWs()
            }
          }
        } 
      }  // END Build
    } 
    stage('dockerBuild') {
      steps {
        script {
          print("build")
          sh"pwd"

          
          if(!(targetProject.equals(EXCLUDE_PROJECT))){
                
            withCredentials([usernamePassword(credentialsId: 'art-service',usernameVariable: 'username',passwordVariable: 'password')]) {
                          
              unstash name:"dockerfile"
              unstash name:"jarfile"
              
              /*
              docker.withRegistry('https://iss-docker-dev.artifactory.molexcloud.com', 'art-service') {
            
                def customImage = docker.build("${IMAGE_ARTIFACT_ID}:${IMAGE_VERSION}_${env.BUILD_ID}")
                customImage.push()
              }
              */
              deployToArtifactory()
            }                
              
          }
        }
      }
    }

    stage('Artifact Scan (X-Ray)') {
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          script {
            if(!(targetProject.equals(EXCLUDE_PROJECT))){
              runXRay()
            }
          }
        }
      }
    }
  
  }
  post { //Declarative: Post Action
    
    success {
      echo 'I succeeeded!'
      updateBuildStatus("SUCCESSFUL", currentBuild.fullDisplayName, settings.PROJECT_DETAILS['hash'], env.RUN_DISPLAY_URL)
      
      sendStatusByMail(settings.PROJECT_DETAILS, settings.PROJECT_REPOS,"SUCCESS", env.RUN_DISPLAY_URL, currentBuild.fullDisplayName, EMAIL_INFO)
    }
    failure {
      echo "Failure"

      updateBuildStatus("FAILED", currentBuild.fullDisplayName,settings.PROJECT_DETAILS['hash'], env.RUN_DISPLAY_URL)

      sendStatusByMail(settings.PROJECT_DETAILS, settings.PROJECT_REPOS, "FAILURE", env.RUN_DISPLAY_URL, currentBuild.fullDisplayName, EMAIL_INFO)
    }
  }
}
