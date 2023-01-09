def ColorMap = [
    'SUCCESS':'good',
    'FAILURE':'danger'
]

//This pipeline should build a docker image and deploy to amazon ecs 
pipeline{
    agent any
    //some environment variables needed by aws 
    environment{
        registryCreds = 'ecr:us-east-1:aws-creds'  
        appRegistry = '374410237047.dkr.ecr.us-east-1.amazonaws.com/vprofileapp-img'
        registryUrl = 'https://374410237047.dkr.ecr.us-east-1.amazonaws.com'
        cluster = 'vprofile'
        service = 'vprofile-app-svc'
    }
    tools{
        maven "MAVEN_3.6.3"
        jdk "JDK-8"
    }
    stages{
        stage('Fetch Code'){
            steps{
                git branch: 'docker', url: 'https://github.com/lite2k/vprofile-project'
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
        stage('Build Docker Image'){
            steps{
                script{
                    dockerImage = docker.build(appRegistery + ":$BUILD_NUMBER","./Docker-files/app/multistage/")
                }
            }   
        }
        stage('Upload Image to Amazon ECR'){
            steps{
                script{
                    docker.withRegistry(registryUrl, registryCreds){
                        dockerImage.push("$BUILD_NUMBER")
                        dockerImage.push("latest")
                    }
                }
            }
        }
        stage('Deploy to Amazon ECS'){
            steps{
                withAWS(credentials: 'aws-creds', region: 'us-east-1'){
                    sh "aws ecs update-service --cluster ${cluster} --service ${service} --force-new-deployment"
                }
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
