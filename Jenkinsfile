pipeline{
    agent any 

    environment {
        registry = "arbazm10/vprofileapparbaz"
        registryCredential = "dockerhub"
        JAVA_HOME = tool name: 'JAVA_HOME', type: 'JDK'
        PATH = "${env.JAVA_HOME}/bin:${env.PATH}"

    }
    stages {
        stage('Print Available JDKs') {
            steps {
                script {
                    def jdks = tool(name: 'JAVA_HOME', type: 'JDK')
                    echo "Available JDKs: ${jdks}"
                    sh 'echo $JAVA_HOME'
                    sh 'echo $PATH'
                }
            }
        }
        stage('BUILD') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'now Archiving'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }

        }
        stage('UNIT TEST'){
            steps {
                sh 'mvn test'
            }
        }
        stage('INTEGRATION TEST') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }
        stage('CODE ANALYSIS WITH CHECKSTYLE'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }
        stage('CODE ANALYSIS WITH SONARQUBE') {
            environment {
                scannerHome = tool 'mysonarscanner4'
            }
            steps {
                withSonarQubeEnv('sonar-pro') {
                    sh ''' ${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                    -Dsonar.projectName=vprofile-repo \
                    -Dsonar.projectVersion=1.0 \
                    -Dsonar.sources= src \
                    -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                    -Dsonar.junit.reportsPath=target/surefire-reports/ \
                    -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                    -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Build App Image') {
            steps{
                script {
                    dockerImage = docker.build registry + ":V$BUILD_NUMBER"
                }
            }
        }
        stage('Upload Image'){
            steps{
                script {
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push("V$BUILD_NUMBER")
                        dockerImage.push('latest')
                    }
                }
            }
        }
        stage('Remove Unused docker Image') {
            steps {
                sh "docker rmi $registry:V$BUILD_NUMBER"
            }
        }
        stage('Kubernetes Deploy') {
            agent {
                label 'AKS1'
            }
            
            steps {
                sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
                }
            
        }
    }

}