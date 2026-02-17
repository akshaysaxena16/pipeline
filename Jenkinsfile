aws stepfunctions list-state-machines --region "$AWS_REGION" \
  --query "stateMachines[?name=='dummy-cert-test'].stateMachineArn | [0]" \
  --output text
pipeline {
    agent { docker { image 'python:3.5.1' } }
    stages {
        stage('build') {
            steps {
                sh 'python --version'
            }
        }
    }
}
