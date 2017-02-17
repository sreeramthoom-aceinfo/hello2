
properties([
  [$class: 'ParametersDefinitionProperty', parameterDefinitions: [
    [$class: 'StringParameterDefinition', defaultValue: 'master', description: 'The git branch to build', name: 'GIT_BRANCH'],
    [$class: 'StringParameterDefinition', defaultValue: 'latest', description: 'The docker image tag', name: 'IMAGE_TAG']
  ]]
])

node('master') {
    wrap([$class: 'BuildUser']) {
    def oc = "oc"
    def osHost = "ocpc.gitook.com:8443"
    def osCredentialId = 'OpenshiftCredentialId'
    def gitUrl = 'https://github.com/aceinfo-jenkins/hello2.git'
    def gitCredentialId = 'jenkinsGithubCredentialId'
    def nexusRegistry = "nexus.gitook.com:8447/demouser"
    def nexusCredentialId = '41aebb46-b195-4957-bae0-78376aa149b0'
    def devTemplate = "osTemplate.yaml"

    stage ('Git Pull') {
      checkout([$class: 'GitSCM',
        branches: [[name: "${GIT_BRANCH}"]],
        doGenerateSubmoduleConfigurations: false,
        extensions: [],
        submoduleCfg: [],
        userRemoteConfigs: [[credentialsId: gitCredentialId, url: gitUrl]]
      ])
    }

    stage ('Build and Test') {
      dir("${env.WORKSPACE}") {
        sh """
          ./gradlew clean build
        """
      }
    }

    stage ('Build Docker Image') {
      input message: "Continue Build Docker Image?", ok: "Build"
      dir("${env.WORKSPACE}") {
        sh """
           ./gradlew buildDocker
        """
      }
    }

    stage ('Push Docker Image to Repository') {
      input message: "Push Docker Image to Repository?", ok: "Push"
      withCredentials([
        [$class: 'UsernamePasswordMultiBinding', credentialsId: nexusCredentialId, usernameVariable: 'NEXUS_USERNAME', passwordVariable: 'NEXUS_PASSWORD']
      ]) {
        dir("${env.WORKSPACE}") {
          sh """
             docker login nexus.gitook.com:8447 --username ${env.NEXUS_USERNAME} --password ${env.NEXUS_PASSWORD}
             docker tag demouser/hello2:latest nexus.gitook.com:8447/demouser/hello2:${IMAGE_TAG}
             docker push nexus.gitook.com:8447/demouser/hello2:${IMAGE_TAG}
             docker rmi demouser/hello2:latest
          """
        }
      }
    }

    stage ('Deploy to Openshift') {
      input message: "Deploy Image to Openshift?", ok: "Deploy"
      withCredentials([
        [$class: 'UsernamePasswordMultiBinding', credentialsId: osCredentialId, usernameVariable: 'OS_USERNAME', passwordVariable: 'OS_PASSWORD'],
        [$class: 'UsernamePasswordMultiBinding', credentialsId: nexusCredentialId, usernameVariable: 'NEXUS_USERNAME', passwordVariable: 'NEXUS_PASSWORD']
      ]) {
        sh """
          ${oc} login ${osHost} --username=${env.OS_USERNAME} --password=${env.OS_PASSWORD} --insecure-skip-tls-verify
        """

        try {
          sh """
            ${oc} project ${env.BUILD_USER_ID}
          """         
        } catch (Exception e) {
          sh """
            ${oc} new-project ${env.BUILD_USER_ID} --display-name="${env.BUILD_USER_ID}'s Development Environment"
            ${oc} secrets new-dockercfg "nexus-${env.BUILD_USER_ID}" --docker-server=${nexusRegistry} \
              --docker-username="${env.NEXUS_USERNAME}" --docker-password="${env.NEXUS_PASSWORD}" --docker-email="docker@gitook.com"
            ${oc} secrets link default "nexus-${env.BUILD_USER_ID}" --for=pull
            ${oc} secrets link builder "nexus-${env.BUILD_USER_ID}" --for=pull
            ${oc} secrets link deployer "nexus-${env.BUILD_USER_ID}" --for=pull
          """                
        }

        try {
          sh """
            ${oc} process -f ${devTemplate} | ${oc} replace --force  -f - -n ${env.BUILD_USER_ID}
          """         
        } catch (Exception e) {
          sh """
            ${oc} process -f ${devTemplate} | ${oc} create -f - -n ${env.BUILD_USER_ID}
          """                
        }

        sh """
          ${oc} tag --source=docker ${nexusRegistry}/hello2:${IMAGE_TAG} ${env.BUILD_USER_ID}/hello2-is:latest --insecure
          sleep 5
          ${oc} import-image hello2-is --confirm --insecure | grep -i "successfully"

          echo "Liveness check URL: http://`${oc} get route hello2-rt -n ${env.BUILD_USER_ID} -o jsonpath='{ .spec.host }'`/hello-world"
        """
      }
    }
  
    stage ('Destroy Openshift Environment') {
      input message: "Delete Openshift Environment(Cleanup)?", ok: "Delete"

      withCredentials([
        [$class: 'UsernamePasswordMultiBinding', credentialsId: osCredentialId, usernameVariable: 'OS_USERNAME', passwordVariable: 'OS_PASSWORD']
      ]) {
        sh """
          ${oc} login ${osHost} --username=${env.OS_USERNAME} --password=${env.OS_PASSWORD} --insecure-skip-tls-verify
          ${oc} delete project ${env.BUILD_USER_ID}
          ${oc} logout
        """
      }
    }
  }
}
