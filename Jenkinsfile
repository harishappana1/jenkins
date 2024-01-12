pipeline {
    agent any
    stages {
        stage('Deploy Image') {
            steps {
                script {
                    properties([
                        parameters([
                            choice(name: 'ENVIRONMENT', choices: ['test', 'prod'], description: 'Select the environment')
                        ])
                    ])
                    def credentialsId = (params.ENVIRONMENT == 'test') ? 'devsecops-nonprod-aws-credentials' : 'devsecops-prod-aws-credentials'
                    def taskDefName = (params.ENVIRONMENT == 'test') ? 'test-clamav' : 'prod-clamav'
                    def ecsClusterName = (params.ENVIRONMENT == 'test') ? 'test-clamav-ecs-cluster' : 'prod-clamav-ecs-cluster'
                    def serviceName = 'clamav'

                    deployToEnvironment(credentialsId, taskDefName, ecsClusterName, serviceName)
                }
            }
        }
    }
}
