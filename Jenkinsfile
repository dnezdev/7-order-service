namespace = "production"
serviceName = "jobber-order"
service = "Jobber Order"

def groovyMethods

m1 = System.currentTimeMillis()

pipeline {
  agent {
    label 'Jenkins-Agent'
  }

  tools {
    nodejs "NodeJS"
    dockerTool "Docker"
  }

  environment {
    DOCKER_CREDENTIALS = credentials("dockerhub")
    IMAGE_NAME = "dnezdev" + "/" + "jobber-order"
    IMAGE_TAG = "stable-${BUILD_NUMBER}"
  }

  stages {
    stage("Cleanup Workspace") {
      steps {
        cleanWs()
      }
    }

    stage("Prepare Environment") {
      steps {
        sh "[ -d pipeline ] || mkdir pipeline"
        dir("pipeline") {
          // Add your jenkins automation url to url field
          git branch: 'main', credentialsId: 'github', url: 'https://github.com/dnezdev/10-jenkins-automation'
          script {
            groovyMethods = load("functions.groovy")
          }
        }
        // Add your order github url to url field
        git branch: 'main', credentialsId: 'github', url: 'https://github.com/dnezdev/7-order-service'
        sh 'npm install'
      }
    }

    stage("Lint Check") {
      steps {
        sh 'npm run lint:check'
      }
    }

    stage("Code Format Check") {
      steps {
        sh 'npm run prettier:check'
      }
    }

    stage("Unit Test") {
      steps {
        sh 'npm run test'
      }
    }

     // stage("Build and Push") {
    //   steps {
    //     sh "docker login -u $DOCKERHUB_CREDENTIAL_USR --password $DOCKERHUB_CREDENTIALS_PSW"
    //     sh "docker build -t $IMAGE_NAME ."
    //     sh "docker tag $IMAGE_NAME $IMAGE_NAME:$IMAGE_TAG"
    //     sh "docker tag $IMAGE_NAME $IMAGE_NAME:stable"
    //     sh "docker push $IMAGE_NAME:$IMAGE_TAG"
    //     sh "docker push $IMAGE_NAME:stable"
    //   }
    // }

    stage("Build and Push") {
    steps {
        script {
            try {
                // Log in to Docker Hub
                sh "docker login -u $DOCKERHUB_CREDENTIAL_USR --password $DOCKERHUB_CREDENTIALS_PSW"

                // Build the Docker image
                sh "docker build -t $IMAGE_NAME ."

                // Tag the Docker image
                sh "docker tag $IMAGE_NAME $IMAGE_NAME:$IMAGE_TAG"
                sh "docker tag $IMAGE_NAME $IMAGE_NAME:stable"

                // Push the Docker image to Docker Hub
                sh "docker push $IMAGE_NAME:$IMAGE_TAG"
                sh "docker push $IMAGE_NAME:stable"

                // Log success message
                echo "Docker image build and push completed successfully."
            } catch (Exception e) {
                // Log error message if any step fails
                echo "Error occurred during Docker image build and push: ${e.message}"
                currentBuild.result = 'FAILURE' // Mark the build as failed
            } finally {
                // Always logout from Docker Hub to clear credentials
                sh "docker logout"
            }
        }
    }
}
//--------------------------------------------------------

    // stage("Clean Artifacts") {
    //   steps {
    //     sh "docker rmi $IMAGE_NAME:$IMAGE_TAG"
    //     sh "docker rmi $IMAGE_NAME:stable"
    //   }
    // }
    stage("Create New Pods") {
      steps {
        withKubeCredentials(kubectlCredentials: [[
                                                  caCertificate: '',
                                                  clusterName: 'minikube',
                                                  contextName: 'minikube',
                                                  credentialsId: 'jenkins-k8s-token',
                                                  namespace: '',
                                                  serverUrl: 'https://172.20.97.179:8443']]){
          script {
            def pods = groovyMethods.findPodsFromName("${namespace}", "${serviceName}")
            for (podName in pods) {
              sh """
                kubectl delete -n ${namespace} pod ${podName}
                sleep 10s
              """
            }
          }
        }
      }
    }
  }
  post {
    success {
      script {
        m2 = System.currentTimeMillis()
        def durTime = groovyMethods.durationTime(m1, m2)
        def author = groovyMethods.readCommitAuthor()
        groovyMethods.notifySlack("", "jobber-jenkins", [
        				[
        					title: "BUILD SUCCEEDED: ${service} Service with build number ${env.BUILD_NUMBER}",
        					title_link: "${env.BUILD_URL}",
        					color: "good",
        					text: "Created by: ${author}",
        					"mrkdwn_in": ["fields"],
        					fields: [
        						[
        							title: "Duration Time",
        							value: "${durTime}",
        							short: true
        						],
        						[
        							title: "Stage Name",
        							value: "Production",
        							short: true
        						],
        					]
        				]
        		]
        )
      }
    }
    failure {
      script {
        m2 = System.currentTimeMillis()
        def durTime = groovyMethods.durationTime(m1, m2)
        def author = groovyMethods.readCommitAuthor()
        groovyMethods.notifySlack("", "jobber-jenkins", [
        				[
        					title: "BUILD FAILED: ${service} Service with build number ${env.BUILD_NUMBER}",
        					title_link: "${env.BUILD_URL}",
        					color: "error",
        					text: "Created by: ${author}",
        					"mrkdwn_in": ["fields"],
        					fields: [
        						[
        							title: "Duration Time",
        							value: "${durTime}",
        							short: true
        						],
        						[
        							title: "Stage Name",
        							value: "Production",
        							short: true
        						],
        					]
        				]
        		]
        )
      }
    }
  }
}
