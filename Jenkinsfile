def initialise() {
    cleanWs()
    checkout scm
}

def AwsAuthenticate() {

    withCredentials([usernamePassword(credentialsId: "", passwordVariable: '', usernameVariable: '')]) {
        sh """
            curl https://github.com/raw/CommercialCloud-EAC/python-scripts/master/aws_cli_saml_ping_v2/requirements.txt > ./requirements.txt
            pip3 install -Ur requirements.txt
            ROLE=''
            curl https://github.com/raw/CommercialCloud-EAC/python-scripts/master/aws_cli_saml_ping_v2/authenticate_py3.py > ./authenticate.py
            curl https://github.com/raw/CommercialCloud-EAC/python-scripts/master/aws_cli_saml_ping_v2/prod.cer > ./prod.cer
            mkdir -p /home/jenkins/.aws/
            touch /home/jenkins/.aws/credentials
            python3 authenticate.py -u ${} -p '${}' -r \$ROLE
          
            echo "check out credential"
            cat /home/jenkins/.aws/credentials
        """
    }
}

pipeline {
    environment
    {
        JAVA_VERSION            = "1.8.0"
        PYTHON_VERSION          = "3.6"
        SCALA_VERSION           = "scala-2.11"
    }

    agent
    {
        node
        {
            label 'python'
        }
    }
    
    options
    {
        disableConcurrentBuilds()
        skipStagesAfterUnstable()
    }
    
  parameters {
    string(name: 'S3_BUCKET', defaultValue: "bucket-to-upload-artifacts", description: "Artifact's S3 directory")
    booleanParam(name: 'SKIP_UNIT_TESTS', defaultValue: false, description: "Skips running Maven tests. Use only when deploying code ")
  }
  
  stages {

      stage ('Build Longitudinal data pipeline') {
          steps {
                initialise()
                script {
                    withMaven(maven:'Maven3', mavenSettingsConfig:'', jdk:'JDK8') {
                        sh label: 'building jar', script: """
                        mvn clean install -Dmaven.test.skip=${params.SKIP_UNIT_TESTS}
                    """
                    }
                }
            }
        }
    stage('Deploy artifacts to S3 directory'){

        when {
          // Deploy if branch is neither master nor release
          not{
            expression {
              env.BRANCH_NAME == "master"
            }
          }
        }

        steps {
            script {
                AwsAuthenticate()

                sh """
                    export AWS_PROFILE=saml
                    export PYTHON_VERSION=3.6
                    /home/jenkins/.local/bin/aws s3 cp pipeline/target/ s3://${params.S3_BUCKET}${env.BRANCH_NAME}/ --recursive --exclude "*" --include "*dependencies.jar" --sse AES256
                """
            }
        }
      }
  }
  
  post {
    always {
      echo 'This will always run'
      emailext body:  "Build URL: ${BUILD_URL}. Artifacts deployed to S3 directory: s3://${params.S3_BUCKET}${env.BRANCH_NAME}/",
        subject: "$currentBuild.currentResult-$JOB_NAME",
        to: 'DataEngineers_DL@.com'
    }
    success {
      echo 'Build completed successfully'
    }
    failure {
      echo 'This will run only if failed'
      echo ''
    }
    unstable {
      echo 'This will run only if the run was marked as unstable'
    }
    changed {
      echo 'This will run only if the state of the Pipeline has changed'
      echo 'For example, if the Pipeline was previously failing but is now successful'
    }
  }
}
