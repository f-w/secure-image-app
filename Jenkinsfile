
import groovy.json.JsonOutput

def APP_NAME = 'secure-image-api'
def BUILD_CONFIG = APP_NAME
def IMAGESTREAM_NAME = APP_NAME
def TAG_NAMES = ['dev', 'test', 'prod']
def CMD_PREFIX = 'PATH=$PATH:$PWD/node-v8.9.4-linux-x64/bin'
def NODE_URI = 'https://nodejs.org/dist/v8.9.4/node-v8.9.4-linux-x64.tar.xz'
def PIRATE_ICO = 'http://icons.iconarchive.com/icons/aha-soft/torrent/64/pirate-icon.png'
def JENNKINS_ICO = 'https://wiki.jenkins-ci.org/download/attachments/2916393/logo.png'

def notifySlack(text, channel, url, attachments, icon) {
    def slackURL = url
    def jenkinsIcon = icon
    def payload = JsonOutput.toJson([text: text,
        channel: channel,
        username: "Jenkins",
        icon_url: jenkinsIcon,
        attachments: attachments
    ])
    sh "curl -s -S -X POST --data-urlencode \'payload=${payload}\' ${slackURL}"
}

node {
  stage('Checkout') {
    echo "Checking out source"
    checkout scm
  }
  
  stage('Install') {
    echo "Setup: ${BUILD_ID}"
    
    // The version of node in the `node` that comes with OpenShift is too old
    // so I use a generic Linux and install my own node from LTS.
    sh "curl ${NODE_URI} | tar -Jx"

    // setup the node dev environment
    sh "${CMD_PREFIX} npm i --only=dev"
    // not sure if this needs to be added to package.json.
    sh "${CMD_PREFIX} npm i escape-string-regexp"
    sh "${CMD_PREFIX} npm -v"
    sh "${CMD_PREFIX} node -v"
  }
  
  stage('Test') {
    echo "Testing: ${BUILD_ID}"
    // Run a security check on our packages
    try {
      sh "${CMD_PREFIX} npm run test:security"
    } catch (error) {
      echo "NSP ERROR\n ${error}"
      notifySlack("NSP Security Warning #${BUILD_ID}\n${error}", "#secure-image-app", "https://hooks.slack.com/services/${SLACK_TOKEN}", [], PIRATE_ICO)
    }

    // Run our unit tests et al.
    sh "${CMD_PREFIX} npm test"
  }

  stage('Build') {
    echo "Build: ${BUILD_ID}"
    // run the oc build to package the artifacts into a docker image
    openshiftBuild bldCfg: 'secure-image-api', showBuildLogs: 'true', verbose: 'true'

    // Don't tag with BUILD_ID so the pruner can do it's job; it won't delete tagged images.
    // Tag the images for deployment based on the image's hash
    IMAGE_HASH = sh (
      script: """oc get istag ${IMAGESTREAM_NAME}:latest -o template --template=\"{{.image.dockerImageReference}}\"|awk -F \":\" \'{print \$3}\'""",
      returnStdout: true).trim()
    echo ">> IMAGE_HASH: ${IMAGE_HASH}"

    openshiftTag destStream: IMAGESTREAM_NAME, verbose: 'true', destTag: TAG_NAMES[0], srcStream: IMAGESTREAM_NAME, srcTag: "${IMAGE_HASH}"

    notifySlack("Build Completed #${BUILD_ID}\nGo to OpenShift to promote this image.", "#secure-image-app", "https://hooks.slack.com/services/${SLACK_TOKEN}", [], JENNKINS_ICO)
  }
}

stage('Test Promotion') {
  timeout(time: 1, unit: 'DAYS') {
    input message: "Deploy to test?", submitter: 'jleach-admin'
  }

  node {
    stage('Promotion') {
      openshiftTag destStream: IMAGESTREAM_NAME, verbose: 'true', destTag: TAG_NAMES[1], srcStream: IMAGESTREAM_NAME, srcTag: "${IMAGE_HASH}"
    }
  }
}
