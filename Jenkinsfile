def registry = 'https://solodevops1.jfrog.io'
def imageName = 'solodevops1.jfrog.io/valaxy-docker-local/ttrend'
def version   = '2.1.5'
pipeline {
    agent {
        node {
            label 'maven'
        }
    }
environment {
    PATH="/opt/apache-maven-3.9.4/bin:$PATH"
}

    stages {
        stage('build') {
            steps {
                echo ".... Build started...."
                sh 'mvn clean deploy -Dmaven.test.skip=true'
                echo ".... Build completed...."
            }
        }
        stage('test'){
            steps {
                echo ".... Unit test started...."
                sh 'mvn surefire-report:report'
                echo ".... Unit test completed...."
            }
        }
   
      stage('SonarQube analysis') {
     environment {
       scannerHome = tool 'solo01-sonar-scanner';
    }
     steps {
   withSonarQubeEnv('sonarqube-server') { // If you have configured more than one global server connection, you can specify its name
   sh "${scannerHome}/bin/sonar-scanner"
   }
   }
  }
  stage('Quality Gate') {
    steps {
        script {
  timeout(time: 1, unit: 'HOURS') { // Just in case something goes wrong, pipeline will be killed after a timeout
    def qg = waitForQualityGate() // Reuse taskId previously collected by withSonarQubeEnv
    if (qg.status != 'OK') {
      error "Pipeline aborted due to quality gate failure: ${qg.status}"
    }
   }
 }
    }
    }
         stage("Jar Publish") {
        steps {
            script {
                    echo '<--------------- Jar Publish Started --------------->'
                     def server = Artifactory.newServer url:registry+"/artifactory" ,  credentialsId:"artifact-cred"
                     def properties = "buildid=${env.BUILD_ID},commitid=${GIT_COMMIT}";
                     def uploadSpec = """{
                          "files": [
                            {
                              "pattern": "jarstaging/(*)",
                              "target": "libs-release-local/{1}",
                              "flat": "false",
                              "props" : "${properties}",
                              "exclusions": [ "*.sha1", "*.md5"]
                            }
                         ]
                     }"""
                     def buildInfo = server.upload(uploadSpec)
                     buildInfo.env.collect()
                     server.publishBuildInfo(buildInfo)
                     echo '<--------------- Jar Publish Ended --------------->'  
            
            }
        }   
    }

    stage(" Docker Build ") {
      steps {
        script {
           echo '<--------------- Docker Build Started --------------->'
           app = docker.build(imageName+":"+version)
           echo '<--------------- Docker Build Ends --------------->'
        }
      }
    }

            stage (" Docker Publish "){
        steps {
            script {
               echo '<--------------- Docker Publish Started --------------->'  
                docker.withRegistry(registry, 'artifact-cred'){
                    app.push()
                }    
               echo '<--------------- Docker Publish Ended --------------->'  
            }
        }
    }
stage(" Deploy via Helm") {
       steps {
         script {
            echo '<--------------- Helm Deploy Started --------------->'
            sh 'helm install ttrend ttrend-0.1.0.tgz'
            echo '<--------------- Helm deploy Ends --------------->'
         }
       }
     }
}
}