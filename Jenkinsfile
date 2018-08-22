import java.text.SimpleDateFormat

def argocdAppPrefix = "hello-world-ks"
def deployable_branches = ["master"]
def ptNameVersion = "${argocdAppPrefix}-${UUID.randomUUID().toString().toLowerCase()}"
def imageName = "argoprojdemo/argo-cd-hello-world-app"
def deployOrg = "argoproj"
def deployRepo = "argo-cd-hello-world-config"
def deployRepoUrl = "git@github.com:${deployOrg}/${deployRepo}.git"
def argocdServer = "argo-cd-demo.argoproj.io"
def appWaitTimeout = 600

podTemplate(name: ptNameVersion, label: ptNameVersion, containers: [
    containerTemplate(name: 'builder', image: 'argoprojdemo/argo-cd-hello-world-ci-builder:latest', ttyEnabled: true, command: 'cat', args: ''),
    containerTemplate(name: 'docker', image: 'docker:17.09', ttyEnabled: true, command: 'cat', args: '' ),
    containerTemplate(name: 'argo-cd-tools', image: 'argoproj/argo-cd-tools:latest', ttyEnabled: true, command: 'cat', args: '', envVars:[envVar(key: 'GIT_SSH_COMMAND', value: 'ssh -o StrictHostKeyChecking=no')] ),
    containerTemplate(name: 'argo-cd-cli', image: 'argoproj/argocd-cli:v0.7.1', ttyEnabled: true, command: 'cat', args: '' ),
    ],
    volumes: [hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')]
  )

{
    // DO NOT CHANGE
    def isPR = env.CHANGE_ID != null
    def branch = env.CHANGE_ID != null ? env.CHANGE_TARGET : env.BRANCH_NAME
    def dateFormat = new SimpleDateFormat("yyyyMMddHHmm")
    def date = new Date()
    def date_tag = dateFormat.format(date)

    // exit gracefully if not the master branch (or, rather, not in deployable_branches)
    if (!deployable_branches.contains(branch)) {
        stage("Skipping pipeline") {
            println "Branch: ${branch} is not part of deployable_branches"
            println "Skipping pipeline"
        }
        currentBuild.result = 'SUCCESS'
        return
    }

    node(ptNameVersion) {
        // DO NOT CHANGE
        def scmInfo = checkout scm
        def gitCommit = "${scmInfo.GIT_COMMIT}"
        tag = "${env.BUILD_TAG}-${gitCommit}"

        // Build Stage
        stage('Build') {
            withCredentials([usernamePassword(credentialsId: "docker-credentials", passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                container('builder') {
                    sh "docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD"
                    sh "make publish"
                }
            }
        }
        def env = "preprod"
        stage( "Deploy ${env}" ) {
            container('argo-cd-tools') {
                println("Deploying to ${argocdAppPrefix}")
                dir("deployment-${env}-${tag}") {
                    withCredentials([file(credentialsId: 'githubDeployKey', variable: 'GIT_DEPLOY_KEY')]) {
                        sh "mkdir /root/.ssh/ && cp \$GIT_DEPLOY_KEY /root/.ssh/id_rsa && chmod 400 /root/.ssh/id_rsa"
                        sh "git clone ${deployRepoUrl}"
                        sh "cd ${deployRepo} && ./update-image.sh ${env} ${argocdAppPrefix} ${imageName} ${deployOrg} ${deployRepo} ${gitCommit}"
                    }
                }          
            }
            container('argo-cd-cli') {
                withCredentials([string(credentialsId: "argocdAuthToken", variable: 'ARGO_CD_TOKEN')]) {
                    sh "/argocd --auth-token $ARGO_CD_TOKEN --server ${argocdServer} app sync ${argocdAppPrefix}-${env}"
                    sh "/argocd --auth-token $ARGO_CD_TOKEN --server ${argocdServer} app wait ${argocdAppPrefix}-${env} --timeout ${appWaitTimeout}"
                }
            }
        }
    }
}
