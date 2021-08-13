import groovy.json.JsonOutput
import groovy.json.JsonSlurperClassic
import com.cloudbees.groovy.cps.NonCPS

def changedObj
def deletedObj
def secrets
def vaultUrl = System.getenv("VAULT_URL")
def vaultPath = System.getenv("VAULT_PATH")

@NonCPS
def jsonParse(def json) {
    new groovy.json.JsonSlurperClassic().parseText(json)
}
@NonCPS
def List<String> getChangedFilesList() {
    changedFiles = []
    for (changeLogSet in currentBuild.changeSets) { 
        for (entry in changeLogSet.getItems()) { 
            for (file in entry.getAffectedFiles()) {
                changedFiles.add(file.getPath())
            }
        }
    }
    return changedFiles
}
def getDeploymentEnvironment() {
    if (env.BRANCH_NAME.startsWith('PR-')) {
        return 'development'
    } else if (env.BRANCH_NAME == 'master') {
        return 'production'
    }
    return env.BRANCH_NAME
}
pipeline {
    options {
        buildDiscarder(logRotator(numToKeepStr:'10'))
        timestamps()
        ansiColor('xterm')
    }
    agent {
        label 'docker-v20.10'
    }
    stages {
        stage ('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Get Changed Files') {
            steps {
                script {
                    def changedFiles = getChangedFilesList()
                    def allmaps = [:]
                    def count = 1
                    for (file in changedFiles) {
                        def submap = [:]
                        def contents = readFile(file)
                        def file_name = ["filename":"${file}"]
                        def file_contents = ["contents":"${contents}"]
                        submap = file_name + file_contents
                        allmaps[count] = submap
                        count += 1
                    }
                    changedObj = JsonOutput.toJson(allmaps)
                    echo changedObj
                }
            }
        }
        stage('Vault') {
            steps {
                script {
                    secrets = vaultGetSecrets()
                }
            }
        }
        stage('Write Updated File to Snaplogic') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'maya-hanck-patch-1') {
                        def post = new URL(System.getenv("SNAPLOGIC_URL")).openConnection();
                        post.setRequestMethod("POST");
                        post.setDoOutput(true);
                        post.setRequestProperty("Authorization", "Bearer ${secrets.snaplogicExprLibToken}")
                        post.setRequestProperty("Accept", "application/json");
                        post.setRequestProperty("Content-Type", "application/json");
                        post.getOutputStream().write(changedObj.getBytes("UTF-8"));
                        def postRC = post.getResponseCode();
                        println(postRC);
                    }
                }
            }
        }
    }
}
