@Library('semantic_releasing')_

node {
    def mvnHome = tool 'M3'
    env.PATH = "${mvnHome}/bin/:${env.PATH}"
    properties([
            buildDiscarder(
                    logRotator(artifactDaysToKeepStr: '',
                            artifactNumToKeepStr: '',
                            daysToKeepStr: '',
                            numToKeepStr: '30'
                    )
            ),
            pipelineTriggers([])
    ])
    cleanWs()

    stage('checkout & unit tests & build') {
        git url: "https://github.com/khinkali/KeycloakEventProvider"
        withCredentials([usernamePassword(credentialsId: 'nexus', passwordVariable: 'NEXUS_PASSWORD', usernameVariable: 'NEXUS_USERNAME')]) {
            sh 'mvn -s settings.xml clean package'
        }
        junit allowEmptyResults: true, testResults: '**/target/surefire-reports/TEST-*.xml'
    }

    stage('build image & git tag & docker push') {
        env.VERSION = semanticReleasing()
        currentBuild.displayName = env.VERSION

        sh "mvn versions:set -DnewVersion=${env.VERSION}"
        sh "git config user.email \"jenkins@khinkali.ch\""
        sh "git config user.name \"Jenkins\""
        sh "git tag -a ${env.VERSION} -m \"${env.VERSION}\""
        withCredentials([usernamePassword(credentialsId: 'github', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
            sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/khinkali/KeycloakEventProvider.git --tags"
        }

        sh "docker build -t khinkali/keycloak:${env.VERSION} ."
        withCredentials([usernamePassword(credentialsId: 'dockerhub', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
            sh "docker login --username ${DOCKER_USERNAME} --password ${DOCKER_PASSWORD}"
        }
        sh "docker push khinkali/keycloak:${env.VERSION}"
    }

    stage('create backup from test') {
        def kct = 'kubectl --kubeconfig /tmp/admin.conf --namespace test'
        def keycloakPods = sh(
                script: "${kct} get po -l app=keycloak",
                returnStdout: true
        ).trim()
        def podNameLine = keycloakPods.split('\n')[1]
        def startIndex = podNameLine.indexOf(' ')
        if (startIndex == -1) {
            return
        }
        def podName = podNameLine.substring(0, startIndex)
        echo "podName: ${podName}"
        sh "${kct} exec ${podName} -- /opt/jboss/keycloak/bin/standalone.sh -Dkeycloak.migration.action=export -Dkeycloak.migration.provider=singleFile -Dkeycloak.migration.file=keycloak-export.json -Djboss.http.port=5889 -Djboss.https.port=5998 -Djboss.management.http.port=5779 &"
        sleep 20
        sh "mkdir keycloakimport"
        sh "${kct} cp ${podName}:/opt/jboss/keycloak-export.json ./keycloakimport/keycloak-export.json"
        sh "${kct} create configmap keycloakimport --from-file=keycloakimport --dry-run -o yaml | ${kct} replace configmap keycloakimport -f -"
    }

    stage('deploy to test') {
        sh "sed -i -e 's/        image: khinkali\\/keycloak:0.0.1/        image: khinkali\\/keycloak:${env.VERSION}/' startup.yml"
        sh "kubectl --kubeconfig /tmp/admin.conf apply -f startup.yml"
    }

    stage('deploy to prod') {
        input(message: 'manuel user tests ok?' )
    }

    stage('create backup from prod') {
        def kc = 'kubectl --kubeconfig /tmp/admin.conf'
        def keycloakPods = sh(
                script: "${kc} get po -l app=keycloak",
                returnStdout: true
        ).trim()
        def podNameLine = keycloakPods.split('\n')[1]
        def startIndex = podNameLine.indexOf(' ')
        if (startIndex == -1) {
            return
        }
        def podName = podNameLine.substring(0, startIndex)
        echo "podName: ${podName}"

        sh "${kc} exec ${podName} -- /opt/jboss/keycloak/bin/standalone.sh -Dkeycloak.migration.action=export -Dkeycloak.migration.provider=singleFile -Dkeycloak.migration.file=keycloak-export.json -Djboss.http.port=5889 -Djboss.https.port=5998 -Djboss.management.http.port=5779 &"
        sleep 20
        sh "rm -rf keycloakimport"
        sh "mkdir keycloakimport"
        sh "${kc} cp ${podName}:/opt/jboss/keycloak-export.json ./keycloakimport/keycloak-export.json"
        sh "${kc} create configmap keycloakimport --from-file=keycloakimport --dry-run -o yaml | ${kc} replace configmap keycloakimport -f -"
    }

    stage('deploy to prod') {
        withCredentials([usernamePassword(credentialsId: 'github-api-token', passwordVariable: 'GITHUB_TOKEN', usernameVariable: 'GIT_USERNAME')]) {
            gitHubRelease(env.VERSION, 'khinkali', 'sink', GITHUB_TOKEN)
        }
        sh "sed -i -e 's/  namespace: test/  namespace: default/' startup.yml"
        sh "sed -i -e 's/    nodePort: 31190/    nodePort: 30190/' startup.yml"
        sh "kubectl --kubeconfig /tmp/admin.conf apply -f startup.yml"
    }
}