def ColorMap = ['SUCCESS':'good','FAILURE':'danger']

pipeline{
    agent any
    tools{
        maven "MAVEN_3.6.3"
        jdk "JDK-8"
    }
    stages{
        stage('Fetch Code'){
            steps{
                git branch: 'vp-rem', url: 'https://github.com/lite2k/vprofile-project'
            }
        }
        stage('Build'){
            steps{
                sh 'mvn install -DskipTests'
            }
            post{
                success{
                    echo "Archiving resulting artifact...."
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }
        stage('Test'){
            steps{
                sh 'mvn test'
            }
        } 
        stage('CheckStyle Analysis'){
            steps{
                sh 'mvn checkstyle:checkstyle'
            }
        }
        stage('SonarQube Analysis'){
            environment{
                scannerHome = tool 'sonar_scanner'
            }
            tools{
                jdk "JDK-11"
            }
            steps{
                withSonarQubeEnv('sonar') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
            }
        }
        stage('SonarQube Quality Gate'){
            steps{
                timeout(time:5, unit:'MINUTES'){
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Upload to Nexus'){
            steps{
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: '127.0.0.1:8081',
                    groupId: 'QA',
                    version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                    repository: 'vprofile-repo',
                    credentialsId: 'nexus-login',
                    artifacts: [[
                    artifactId: 'vproapp',
                    classifier: '',
                    file: 'target/vprofile-v2.war',
                    type: 'war']]
                )
            }
        }
    }
    post{
        always{
            echo 'Sneding Slack notifications.'
            slackSend channel: '#vprofile-cicd',
                color: ColorMap[currentBuild.currentResult],
                message: "${currentBuild.currentResult}: Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n" 
        }
    }
}
