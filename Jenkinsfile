#!groovy
// -*- coding: utf-8; mode: Groovy; -*-
@Library('common-utils') _

properties([
    buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '7', daysToKeepStr: '', numToKeepStr: '7')),
    disableConcurrentBuilds(),
    disableResume()
]);

env.SA                    = "";
env.ARTIFACTORY_URL       = "";
env.DOCKER_REGISTRY       = "";

env.APP_NAME              = "";

env.DOCKER_IMAGE = "${env.DOCKER_REGISTRY}.${ARTIFACTORY_URL}/${env.APP_NAME}";

//slackUtils.sendBuildStatusNotification(channelId: 'XXX') {
    node ('dockerhost') {
        if (branch_name == "main") {
//             env.K8S_CLUSTER      = "";
//             env.K8S_NAMESPACE    = "";
//             env.DOCKER_IMAGE_TAG = env.COMMIT_SHA
//             vaultSecretsToEnv()
//             checkout()
//             buildFrontend()
//             dockerPush()
//             deployK8S()
//             cleanUp()
        }
        else if (branch_name == "staging") {
            env.K8S_CLUSTER      = "";
            env.K8S_NAMESPACE    = "";

            checkout()

            env.DOCKER_IMAGE_TAG = "${env.COMMIT_SHA}-staging";
//             vaultSecretsToEnv()
            runTests()
            buildFrontend()
            dockerPush()
            deployK8S()
            cleanUp()
        }
        else {
            checkout()
            sh "echo NOTHING WILL BE DEPLOYED"
            cleanUp()
        }
    }
// }

def checkout() {
    stage('Checkout sources') {
        checkout scm
        env.COMMIT_SHA = gitUtils.getCommitShaShort()
    }
}

def runTests() {
    stage('Testing & Sonar checking') {
        sh """
            docker-compose -f docker/docker-compose.tests.yml build
        """
    }
}

def buildFrontend() {
    stage('Build') {
        env.DOCKER_IMAGE_WITH_TAG = "${DOCKER_IMAGE}:${DOCKER_IMAGE_TAG}"

        sh """
            docker-compose -f docker/docker-compose.prod.yml build
        """
    }
}

def dockerPush() {
    stage('Docker Push to Artifactory') {
        // Artifactory settings
        def artifactoryServer = Artifactory.newServer(url: "https://${ARTIFACTORY_URL}", credentialsId: env.SA);
        artifactoryServer.connection.timeout = 1200;
        def artifactoryDocker = Artifactory.docker (server: artifactoryServer);
        // Artifactory push frontend
        def buildInfoFrontend = artifactoryDocker.push ("${env.DOCKER_IMAGE_WITH_TAG}", env.DOCKER_REGISTRY);
        artifactoryServer.publishBuildInfo buildInfoFrontend
    }
}

def deployK8S() {
    stage('Deploy to K8S') {
        withCredentials([usernamePassword(credentialsId: "${env.SA}", usernameVariable: "", passwordVariable: "")]) {
            def deployer = docker.image('dockerimage');
            deployer.pull();
            withEnv (["CLUSTER=${env.K8S_CLUSTER}"]) {
                    deployer.inside('-u root') {
                      sh ("get-kubeconfig-basic");
                      sh ("envsubst < .helm/values.yaml > .helm/values_generated.yaml");
                      sh ("mv -f .helm/values_generated.yaml .helm/values.yaml");
                      sh ("helm3 upgrade --install ${env.APP_NAME} .helm --namespace ${env.K8S_NAMESPACE} --set app.frontend.image.name=${env.DOCKER_IMAGE} --set app.frontend.image.tag=${env.DOCKER_IMAGE_TAG}");
                }
            }
        }
    }
}

def cleanUp() {
    stage ('Clean Workspace') {
        sh """
            docker image rm ${env.DOCKER_IMAGE_WITH_TAG} || echo 'No image to clean'
        """
        cleanWs();
    }
}
