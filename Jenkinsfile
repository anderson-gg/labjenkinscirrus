// S3 bucket for staging pipeline
ArtifactoryUrl            = "https://art-bobcat.autodesk.com/artifactory"
ArtifactoryCredentialsId  = "svc_p_ottologin"
ArtifactoryRepo           = "team-cieus-generic"
ArtifactoryRepoPath       = "otto"
TargetForDeployment       = "${ArtifactoryRepo}/${ArtifactoryRepoPath}"
BucketPosts = "ottotestcirrus"

// import Globals

// Global variables
GitCommit = ""
DeploymentSucceeded = false
VersionStrings = []

Artifacts = [:]

//Globals.debug = true


/* Updates the online library for master branch. */
def deploycodes3()
{

  withEnv(["VAULT_ADDR=https://vault.aws.autodesk.com"]) {
    withCredentials([
      usernamePassword(
        credentialsId: 'svc_p_ottologin',
        usernameVariable: 'USERNAME',
        passwordVariable: 'PASSWORD'
      )
    ])
    {
      String sResponse = sh(
        script: """
        vault login -method=ldap username='${USERNAME}' password='${PASSWORD}' > /dev/null
        vault read -format=json aws-sts/des/otto/sts/Vault-Deployer
        """,
        returnStdout: true
      )

      // Uncomment for printing temporary credentials
      // (useful for debugging)
      // echo sResponse

      def v = readJSON text: sResponse

      String accessKey = v.data.access_key
      String secretAccessKey = v.data.secret_key
      String sessionToken = v.data.security_token

      withEnv(
        [
          "AWS_ACCESS_KEY_ID=${accessKey}",
          "AWS_SECRET_ACCESS_KEY=${secretAccessKey}",
          "AWS_SESSION_TOKEN=${sessionToken}"
        ]
      )
      {
        // copy to the bucket
        bat "aws s3 cp --quiet c:\\archive\\bundle.zip s3://${BucketPosts}"
      }
    }
  }
}

def uploadartifact()
{
         if ("$BRANCH_NAME" == "master") {
             def server = Artifactory.newServer url: 'https://art-bobcat.autodesk.com/artifactory/', credentialsId: 'svc_p_ottologin'
             def uploadSpec = """{
                 "files": [
                     {
                         "pattern": "*.zip",
                         "target": "team-cieus-generic/build-tools/"
                     }
                 ]
             }"""
             server.upload(uploadSpec)
         }
}

def downloadartifact()
{
         if ("$BRANCH_NAME" == "master") {
             def server = Artifactory.newServer url: 'https://art-bobcat.autodesk.com/artifactory/', credentialsId: 'svc_p_ottologin'
             def downloadSpec = """{
                 "files": [
                     {
                         "pattern": "bundle.zip",
                         "target": "downloadArtifactory/"
                     }
                 ]
             }"""
             server.download(downloadSpec)
         }
}


def publishPost(String os, String pattern)
{
  def artifact = bash(
    script: "ls ${pattern} | grep -o 'post.*\\.zip'",
    returnStdout: true
  ).trim()

  echo "Artifact (${os}): ${artifact}"

  String suffix = osSuffixes[os]

  if (isDeploymentNeeded(suffix)) {
    def server = Artifactory.newServer(
      url: "${ArtifactoryUrl}",
      credentialsId : "${ArtifactoryCredentialsId}"
    )

    server.upload """{
      "files": [
        {
          "pattern": "${pattern}",
          "target": "${TargetForDeployment}/"
        }
      ]
    }"""
  }

  DeploymentSucceeded = true

  Artifacts[os] = artifact
}

def init()
{
  echo "GitCommit test"
}

pipeline {
    agent none
    parameters {
      string(name: 'Greeting', defaultValue: 'Hello', description: 'How should I greet the world?')
    }
    options {
      timestamps()
      skipDefaultCheckout()
    }
    stages{
      stage('Build') {
        when {
          anyOf {
            branch 'master'
            branch 'PR-*'
          }
        }
        stages   {
          stage('Windows') {
            agent {
              node('aws-win2019-mar-2020')
            }
          stages {
            stage ('Checkout') {
              steps {
                //This command is inherited from Pipeline: SCM Step
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], gitTool: 'Default', submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'svc_p_cirrusgitlogin', url: 'https://git.autodesk.com/dpe/cirrus_jenkin_lab.git']]]) 
              }
            }
            stage ('Init') {
              steps {
                bat 'dir'
                bat 'powershell -Command  Get-ChildItem'
                bat 'powershell -Command  gci -recurse -filter "test123"'
                bat 'powershell -Command  Add-Content -Path test -Value "TheEnd"'
                bat 'powershell -Command  Add-Content -Path test123 -Value "TheEnd"' 
                bat 'powershell -Command  Add-Content -Path sathya\\test -Value "TheEnd"'
                bat 'powershell -Command  Add-Content -Path sathya\\test -Value "TheEnd"'
                bat 'powershell -Command  New-Item -Path c:\\bundle -ItemType directory'
                bat 'powershell -Command  New-Item -Path c:\\archive -ItemType directory'
                bat 'powershell -Command  Copy-item -Force -Recurse -Verbose * c:\\bundle'
                bat 'powershell -Command  Compress-Archive -Path C:\\bundle -DestinationPath c:\\archive\\bundle.zip'
                bat 'powershell -Command  Get-ChildItem c:\\archive'
                bat 'powershell -Command  Get-ChildItem c:\\bundle'
              }
            }            
            stage('Upload the package to Artifactory') {
              steps {
                uploadartifact()
              }
            }

            stage("Download Package from Artifactory"){
              steps{
                downloadartifact()
              }
                
            }        

            stage ('Deploy') {
              steps {
                deploycodes3()
              }
              post {
                success {
                  echo "SUCCESS"
                }
              }
            }
          }            
        }
      }
    }
  }  
}
