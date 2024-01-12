pipeline {
    agent any
    environment {
        registryUrl = "123456789012.dkr.ecr.us-east-1.amazonaws.com" // Replace with your ECR registry URL
        dockerImageName = "your-docker-image-name"
        dockerImageTag = "v1.0" // Manually set the version/tag
        ecrCredentialsId = 'ecr-credentials' // Jenkins credentials ID for ECR authentication
    }

    stages {
        stage('Build Docker Image') {
            steps {
                script {
                    // Build Docker image
                    def dockerImage = docker.build("${registryUrl}/${dockerImageName}:${dockerImageTag}")
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    // Push Docker image to ECR
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY', credentialsId: "${ecrCredentialsId}"]]) {
                        sh "aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${registryUrl}"
                        docker.withRegistry("${registryUrl}", "${ecrCredentialsId}") {
                            sh "docker push ${dockerImageName}:${dockerImageTag}"
                        }
                    }
                }
            }
        }

        stage('Deploy Image') {
            steps {
                script {
                    properties([
                        parameters([
                            choice(name: 'ENVIRONMENT', choices: ['Test', 'Prod'], description: 'Select the environment')
                        ])
                    ])
                    def credentialsId = (params.ENVIRONMENT == 'Test') ? 'devsecops-nonprod-aws-credentials' : 'devsecops-prod-aws-credentials'
                    def taskDefName = (params.ENVIRONMENT == 'Test') ? 'test-clamav' : 'prod-clamav'
                    def ecsClusterName = (params.ENVIRONMENT == 'Test') ? 'test-clamav-ecs-cluster' : 'prod-clamav-ecs-cluster'
                    def serviceName = 'clamav'

                    deployToEnvironment(credentialsId, taskDefName, ecsClusterName, serviceName)
                }
            }
        }
    }
}

def deployToEnvironment(credentialsId, taskDefName, ecsClusterName, serviceName) {
    node('aws-cli') {
        container('aws-cli') {
            def region = 'us-east-1'

            stage('Pull Code from GIT') {
                checkout scm
            }

            stage('Deploy Image') {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY', credentialsId: "${credentialsId}"]]) {
                    def nonProdImage = sh(
                        script: "aws ecs describe-task-definition --task-definition ${taskDefName} --region ${region} --query 'taskDefinition.containerDefinitions[0].image'",
                        returnStdout: true
                    ).trim()

                    def currentTaskDef = sh(
                        script: "aws ecs describe-services --cluster ${ecsClusterName} --services ${serviceName} --region ${region} --query 'services[].taskDefinition[]' --output text",
                        returnStdout: true
                    ).trim()

                    def currentTaskDefString = sh(
                        script: "aws ecs describe-task-definition --region ${region} --task-definition ${currentTaskDef} \
                            --query '{ \
                            containerDefinitions: taskDefinition.containerDefinitions, \
                            family: taskDefinition.family, \
                            volumes: taskDefinition.volumes, \
                            placementConstraints: taskDefinition.placementConstraints, \
                            requiresCompatibilities: taskDefinition.requiresCompatibilities, \
                            executionRoleArn: taskDefinition.executionRoleArn, \
                            taskRoleArn: taskDefinition.taskRoleArn, \
                            networkMode: taskDefinition.networkMode, \
                            memory: taskDefinition.memory, \
                            cpu: taskDefinition.cpu \
                            }'",
                        returnStdout: true
                    ).trim()

                    def currentTaskDefJSON = readJSON text: currentTaskDefString

                    if (nonProdImage != currentTaskDefJSON.containerDefinitions[0].image) {
                        currentTaskDefJSON.containerDefinitions.each { containerDefinition ->
                            containerDefinition.image = nonProdImage
                        }

                        def newTaskDefFileName = "/tmp/${serviceName}-${BUILD_NUMBER}-${System.currentTimeMillis()}.json"
                        writeJSON(file: newTaskDefFileName, json: currentTaskDefJSON)

                        def newTaskDefArn = sh(
                            script: "aws ecs register-task-definition --cli-input-json file://${newTaskDefFileName} --query taskDefinition.taskDefinitionArn --output text",
                            returnStdout: true
                        ).trim()

                        def updatedServiceArn = sh(
                            script: "aws ecs update-service --cluster ${ecsClusterName} --service ${serviceName} --task-definition ${newTaskDefArn} --no-force-new-deployment --query service.serviceArn",
                            returnStdout: true
                        ).trim()

                        echo "Deployment completed for ${params.ENVIRONMENT} environment"
                        echo "Updated service Arn: ${updatedServiceArn}"
                    } else {
                        echo "Images are the same; nothing to update"
                        echo "Current Image in ${params.ENVIRONMENT} = ${nonProdImage}, current prod image = ${currentTaskDefJSON.containerDefinitions[0].image}"
                    }
                }
            }
        }
    }
}
