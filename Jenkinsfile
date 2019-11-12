try {
    timeout(time: 20, unit: 'MINUTES') {
      def appName="petspringapp"
      def appNameRoute="petclinicbluegreen"
      def project="petspringapp"
      def tag="blue"
      def altTag="green"
      def verbose="false"
       node('maven') {
           project = env.PROJECT_NAME
           stage("Check the Live Traffic & Switch"){
             sh "oc get route ${appName} -n petspringapp  -o jsonpath='{ .spec.to.name }' --loglevel=4 > activeservice"
             activeService = readFile('activeservice').trim()
             if (activeService == "${appName}-blue") {
                tag = "green"
                altTag = "blue"
            }//active-service
             sh "oc get route ${appName}-${tag} -n petspringapp -o jsonpath='{ .spec.host }' --loglevel=4 > routehost"
             routeHost = readFile('routehost').trim()
             
           }//intialize
           stage("Git CheckOut"){
             checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/satishchennu1/spring-petclinic-data-jdbc.git']]])  
           }
             
           stage('Build WAR') {
              git url: "https://github.com/satishchennu1/spring-petclinic-data-jdbc.git"
              sh "mvn clean package -Popenshift"
            }
            stage('Unit Test') {
              sh "mvn test"
              step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
            }
           stage('Integration Test') {
              sh "mvn verify"
            }
            stage('Code Coverage using Jacoco') { //
              archive 'target/*.jar'
              step([$class: 'JacocoPublisher', execPattern: '**/target/jacoco.exec'])
             }
            stage('Code Quality Using SonarQube') {
              def mvnHome =  tool name: 'maven', type: 'maven'
              withSonarQubeEnv('SonarQube') { 
              sh "${mvnHome}/bin/mvn sonar:sonar -Dsonar.host.url=http://sonarqube-sonarqubeanalysis.apps.ocppilot.ocpcontainer.com/ -DskipTests=true"
              }
            }
           stage('Build Image') {
             openshift.withCluster() {
                openshift.withProject() {
                   def bld = openshift.startBuild('petspringapp')
                   bld.untilEach {
                     return it.object().status.phase == "Running"
                   }
                   bld.logs('-f')
                }  
             }
           }
           
           stage("Push the Image to Registry"){
            echo "uploading the image to registry"
           }
           stage('Deploy Pod') {
             openshift.withCluster() {
               openshift.withProject() {
                //openshift.tag("${appName}:latest", 'petspringapp':${tag}")
                //return !openshift.selector('dc', 'petspringapp'-${tag}).exists()
                //dc.rollout().latest()
                openshift.tag("${appName}:latest", "${appName}:${tag}")
                def dc = openshift.selector('dc', "${appName}-${tag}")
                dc.rollout().status()
               }
             }
           }
           stage("approval Message") {
              input message: "Test deployment: http://${routeHost}. Approve?", id: "approval"
            }           
           stage("Route Live Traffic New Version") {
              sh "oc set -n petspringapp route-backends ${appName} ${appName}-${tag}=100 ${appName}-${altTag}=0 --loglevel=4"
            }
           stage("Go Live URL"){
             echo "URL for Testing : http://${routeHost}"
           }
         }
    }
 } catch (err) {
    echo "in catch block"
    echo "Caught: ${err}"
    currentBuild.result = 'FAILURE'
    throw err
 }    
