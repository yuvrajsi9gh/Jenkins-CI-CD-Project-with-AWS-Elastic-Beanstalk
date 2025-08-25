pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        git_origin = 'https://github.com/<your-repo>.git'
        aws_region = 'ap-south-1'
        aws_credentials = 'aws-jenks'
        aws_application_name = 'Pipeline-project'
        aws_environment_name = 'Pipeline-project-env'
        aws_root_object = 'app.zip'
        aws_key_prefix = 'Pipeline-project/builds'
    }

    tools { nodejs 'nodejs' }

    stages {
        stage('Source Code from GitHub to Jenkins') {
            steps {
                sh 'env'
                git branch: 'main', url: git_origin
            }
        }

        stage('Building Package from our Source Code') {
            steps {
                sh '''
                    npm install
                    npm run build
                    zip -r app.zip * .[^.]*
                '''
            }
        }

        stage('Application deploying to Elastic Beanstalk') {
            steps {
                step([$class: 'AWSEBDeploymentBuilder',
                    credentialId: aws_credentials,
                    awsRegion: aws_region,
                    applicationName: aws_application_name,
                    environmentName: aws_environment_name,
                    keyPrefix: aws_key_prefix,
                    rootObject: aws_root_object,
                    versionLabelFormat: "-${BUILD_NUMBER}"
                ])
            }
        }
    }
}
