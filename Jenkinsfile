def appName = "birthday-paradox"
def replicas = "1"
def devProject = "f82a0891-af0f-40e2-aba6-941a102ea73f-dev" // TODO: Your dev project name goes here
def testProject = "f82a0891-af0f-40e2-aba6-941a102ea73f-test" // TODO: Your test project name goes here
def prodProject = "f82a0891-af0f-40e2-aba6-941a102ea73f-prod" // TODO: Your prod project name goes here

def skopeoToken
def imageTag

def getVersionFromPom() {
    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
    matcher ? matcher[0][1] : null
}

def skopeoCopy(def skopeoToken, def srcProject, def destProject, def appName, def imageTag) {
    sh """skopeo copy --src-tls-verify=false --src-creds=jenkins:${skopeoToken} \
    --dest-tls-verify=false --dest-creds=jenkins:${skopeoToken} \
    docker://image-registry.openshift-image-registry.svc:5000/${srcProject}/${appName}:${imageTag} \
    docker://image-registry.openshift-image-registry.svc:5000/${destProject}/${appName}:${imageTag}"""
}

def deployApplication(def appName, def imageTag, def project, def replicas) {
    openshift.withProject(project) {
        dir("openshift") {
            def result = openshift.process(readFile(file:"deploy.yaml"), "-p", "APPLICATION_NAME=${appName}", "-p", "IMAGE_TAG=${imageTag}", "-p", "APPLICATION_PROJECT=${project}")
            openshift.apply(result)
        }
        openshift.selector("deployment", appName).scale("--replicas=${replicas}")
    }
}

pipeline {
    agent { label "maven" }
    stages {
        stage("Setup") {
            steps {
                script {
                    openshift.withProject(devProject) {
                        skopeoToken = openshift.raw("sa get-token jenkins").out.trim()
                    }
                    imageTag = getVersionFromPom()
                }
            }
        }
        stage("Build & Test") {
            steps {
                sh "mvn clean package"
            }
        }
        stage("Create Image") {
            steps {
                script {
                    openshift.withProject(devProject) {
                        dir("openshift") {
                            def result = openshift.process(readFile(file:"build.yaml"), "-p", "APPLICATION_NAME=${appName}", "-p", "IMAGE_TAG=${imageTag}")
                            openshift.apply(result)
                        }
                        dir("target") {
                            openshift.selector("bc", appName).startBuild("--from-file=${appName}-${imageTag}.jar").logs("-f")
                        }
                    }
                }
            }
        }
        stage("Deploy Application to Dev") {
            steps {
                script {
                    deployApplication(appName, imageTag, devProject, replicas)
                }
            }
        }
        stage("Copy Image to Test") {
            agent { label "jenkins-agent-skopeo" }
            steps {
                script {
                    skopeoCopy(skopeoToken, devProject, testProject, appName, imageTag)
                }
            }
        }
        stage("Deploy Application to Test") {
            steps {
                script {
                    deployApplication(appName, imageTag, testProject, replicas)
                }
            }
        }
        stage("Prompt for Prod Approval") {
            steps {
                input "Deploy to prod?"
            }
        }
        stage("Copy image to Prod") {
            agent { label "jenkins-agent-skopeo" }
            steps {
                script {
                    skopeoCopy(skopeoToken, devProject, prodProject, appName, imageTag)
                }
            }
        }
        stage("Deploy Application to Prod") {
            steps {
                script {
                    deployApplication(appName, imageTag, prodProject, replicas)
                }
            }
        }
    }
}