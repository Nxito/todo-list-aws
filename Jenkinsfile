pipeline {
    agent any
    environment {
        GITHUB_TOKEN = credentials('github_token')
    }
    stages {
        stage('Get Code') {
            //agent { label 'builder' }
            steps {
                echo '[Get Code] Obteniendo código de GitHub...'
                sh 'whoami && hostname && echo ${WORKSPACE}'
                git  branch: 'develop',
                    url: 'https://github.com/Nxito/todo-list-aws.git',
                    credentialsId: 'github_token'
                sh '''
                    curl -o samconfig.toml https://raw.githubusercontent.com/Nxito/todo-list-aws-config/refs/heads/staging/samconfig.toml
                    cat
                '''
                sh 'ls -a'
                echo "$WORKSPACE"
                stash name: 'my-code'
            }
        }
        stage('Static Test') {
            environment {
                PYTHONPATH = "${WORKSPACE}"
            }
            steps {
                sh 'flake8 --format=pylint src > flake8.out'
                sh 'cat flake8.out'
                sh 'bandit -r src -f custom -o bandit.out --msg-template "{abspath}:{line}: {severity}: {test_id}: {msg}" || true'
                sh 'cat bandit.out'
            }
        }
        stage('SAM') {
            steps {
                sh 'sam build'
                sh 'cat samconfig.toml'
                sh '''
                    sam deploy --config-env staging --no-fail-on-empty-changeset
                '''
            //  sh '''
            //         sam deploy \
            //          --stack-name staging-todo-list-aws \
            //          --s3-bucket aws-sam-cli-managed-default-samclisourcebucket-n8htbdjrnr7w \
            //          --region us-east-1 \
            //          --no-confirm-changeset \
            //          --no-fail-on-empty-changeset
            //     '''
            }
        }

        stage('Rest Tests') {
            environment {
                    PYTHONPATH = "${WORKSPACE}"
                    AWS_REGION = 'us-east-1'
                    STACK_NAME = 'staging-todo-list-aws'
                    DYNAMODB_TABLE = 'todoUnitTestsTable'
            }
            steps {
                //Añado la URL base para que el api la use
                script {
                    catchError(buildResult: 'ABORTED', stageResult: 'FAILURE') {
                        def baseUrl = sh(
                            script: "aws cloudformation describe-stacks --stack-name ${STACK_NAME} --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region ${AWS_REGION} --output text",
                            returnStdout: true
                        ).trim()
                        echo "BaseUrlApi: ${baseUrl}"
                        env.BASE_URL = baseUrl
                    }
                }
                echo "Los test usarán la siguiente url: $BASE_URL"
                sh 'pytest -m api  --junitxml=result-rest.xml test/integration/todoApiTest.py || exit'
                junit testResults: 'result-rest.xml', allowEmptyResults: false, skipPublishingChecks: true
            }
        }
        stage('Promote') {
            when {
                expression {
                    currentBuild.currentResult == 'SUCCESS'
                }
            }
            steps {
                echo 'Subiendo cambios a Master'
                sh '''
                        git checkout master
                        git merge -Xours develop
                        git push "https://${GITHUB_TOKEN}@github.com/Nxito/todo-list-aws.git" master
                '''
            }
        }
    }
    post  {
        always {
            cleanWs()
        }
    }
}
