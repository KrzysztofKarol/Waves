#!/usr/bin/env groovy

/*
This is a Jenkins scripted pipeline to launch integration tests. We use following plugins:
- Extended Choice parameter: https://plugins.jenkins.io/extended-choice-parameter
- Generic Webhook Trigger plugin: https://wiki.jenkins.io/display/JENKINS/Generic+Webhook+Trigger+Plugin

We alse need to 

On the GitHub side navigate to Repository Settings > Webhooks >
- Content type: application/json
- SSL Verification: enabled
- Events: choose the events you want to trigger build on
- Payload URL: https://<jenkins_web_address>/generic-webhook-trigger/invoke?token=wavesPipelineTriggerToken

- generate a personal access token with 'repo' permissions add it to Jenkins secrets and specify ID to 'githubPersonalToken' variable

To set up pipeline in Jenkins: New Item > Pipeline > name it > OK > Scroll to Pipeline pane >
- Definition: Pipeline script from SCM, SCM: Git, Repo: 'https://github.com/wavesplatform/Waves.git',
- Lightweight checkout: disabled. Save settings and launch pipeline
*/

@Library('jenkins-shared-lib')
import devops.waves.*
ut = new utils()
scripts = new scripts()
def buildTasks = [:]
def repo_url = 'https://github.com/wavesplatform/Waves.git'
def branch = false
def testTasks = [:]
def gitCommit
def githubRepo = 'wavesplatform/Waves'
def githubPersonalToken = 'waves-github-token'

properties([
    ut.buildDiscarderPropertyObject('14', '30'),
    parameters([
        ut.stringParameterDefinitionObject('pr_from_ref'),
        ut.choiceParameterObject('branch', scripts.getBranches(repo_url), Boolean.TRUE)
    ]),

    pipelineTriggers([
        [$class: 'GenericTrigger',
        genericVariables: [
            [ key: 'source', value: '$.ref', regexpFilter: 'refs/heads/', defaultValue: '' ],
            [ key: 'push_from_sha', value: '$.after'],
            [ key: 'pr_action', value: '$.action'],
            [ key: 'deleted', value: '$.deleted'],
            [ key: 'pr_from_ref', value: '$.pull_request.head.ref' ],
            [ key: 'pr_from_sha', value: '$.pull_request.head.sha' ]],
        // this is a place where some magic occurs ;)
        regexpFilterText: '$deleted$source$pr_action',
        regexpFilterExpression: 'falsemaster|falseversion.+|opened|reopened|synchronize',
        causeString: "Triggered by GitHub Webhook",
        printContributedVariables: true,
        printPostContent: true,
        token: 'wavesPipelineTriggerToken' ]
    ])
])

stage('Aborting this build'){

    // Here we check if parameter 'branch' does not have any assigned value or it's value
    // is a default one -- '-- Failed to retrieve any data---'
    // In this case we won't proceed
    if (params.branch && params.branch.length() && ! params.branch.contains('--')){
        branch = params.branch
    }

    // If this is a hook with a GitHub pull request event, then we take 'branch'
    // from params.pr_from_ref
    if (params.pr_from_ref && params.pr_from_ref.length()){
        branch = params.pr_from_ref
    }

    if (! branch) {
        echo "Aborting this build. Variable 'branch' not defined. We can't proceed since the git branch is uknown..."
        currentBuild.result = Constants.PIPELINE_ABORTED
        return
    }
    else
        echo "Parameters specified: ${params}"
}

if (currentBuild.result == Constants.PIPELINE_ABORTED){
    return
}

timeout(time:90, unit:'MINUTES') {
    node{
        currentBuild.result = Constants.PIPELINE_SUCCESS
        timestamps {
            wrap([$class: 'AnsiColorBuildWrapper', 'colorMapName': 'XTerm']) {
                try {
                    
                    currentBuild.displayName = "#${env.BUILD_NUMBER} - ${branch}"

                    stage('Checkout') {
                        sh 'env'
                        step([$class: 'WsCleanup'])
                        ut.checkout(branch, repoUrl)
                        gitCommit = ut.shWithOutput("git rev-parse HEAD")
                        stash name: 'sources', includes: '**'
                        ut.setGitHubBuildStatus(buildImageName, githubPersonalToken, gitCommit)
                    }

                    testTasks['Unit tests'] = {
                        node('wavesnode'){
                            stage('Unit tests') {
                                unstash 'sources'
                                try{
                                    ut.sbt '-Dquill.macro.log=false -J-Xms3G -J-Xmx3G -J-XX:+UseConcMarkSweepGC -J-XX:+CMSClassUnloadingEnabled ";coverage;checkPR;coverageReport"'
                                }
                                finally{
                                    sh "tar -czvf test-reports.tar.gz -C target/test-reports/ . || true"
                                    stash name: 'test-reports', includes 'test-reports.tar.gz'
                                }
                            }
                        }
                    }

                    testTasks['Integration Test'] = {
                        node('wavesnode'){
                            withEnv(["SBT_THREAD_NUMBER=7"]) {
                                stage('Integration Test') {
                                    unstash 'sources'
                                    sh """
                                        docker rmi com.wavesplatform/node-it || true
                                        docker ps -a
                                        docker images
                                        docker network ls
                                    """
                                    try{
                                        ut.sbt '-mem 40960 clean node-it/test'    
                                    }
                                    finally{
                                        sh "docker system prune -af --volumes || true"
                                        sh "tar -czvf node-logs.tar.gz -C node-it/target/logs/ . || true"
                                        stash name: 'node-logs', includes 'node-logs.tar.gz'
                                    }
                                }
                            }
                        }
                    }
                    parallel testTasks
                }
                catch (err) {
                    currentBuild.result = Constants.PIPELINE_FAILURE
                    println("ERROR caught")
                    println(err)
                    println(err.getMessage())
                    println(err.getStackTrace())
                    println(err.getCause())
                    println(err.getLocalizedMessage())
                    println(err.toString())
                 }
                finally{
                    unstash 'node-logs'
                    unstash 'test-reports'
                    archiveArtifacts artifacts: 'node-logs.tar.gz, test-reports.tar.gz'
                    ut.notifySlack("jenkins-notifications", currentBuild.result)
                }
            }
        }
    }
}
