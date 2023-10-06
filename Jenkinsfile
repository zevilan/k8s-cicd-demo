pipeline {
    agent {
        kubernetes {
            label 'maven'
        }
    }

    parameters {
        gitParameter name: 'BRANCH_NAME', branch: '', branchFilter: '.*', defaultValue: 'master', description: '请选择要发布的分支', quickFilterEnabled: false, selectedValue: 'NONE', tagFilter: '*', type: 'PT_BRANCH'
        choice(name: 'NAMESPACE', choices: ['devops-dev', 'devops-test', 'devops-prod'], description: '命名空间')
        string(name: 'TAG_NAME', defaultValue: '标签名称，必须以 v 开头，例如：v1、v1.0.0', description: '')
    }

    environment {
        DOCKER_CREDENTIAL_ID = 'harbor-user-pass'
        GIT_REPO_URL = '192.168.113.121:28080'
        GIT_CREDENTIAL_ID = 'gitlab-user-pass'
        GIT_ACCOUNT = 'root' // change me
        KUBECONFIG_CREDENTIAL_ID = '4203b0a5-7351-4fa0-ad31-c54f1f34ef99'
        REGISTRY = '192.168.113.122:8858'
        DOCKERHUB_NAMESPACE = 'wolfcode' // change me
        APP_NAME = 'k8s-cicd-demo'
        SONAR_SERVER_URL = 'http://192.168.113.120:31377'
        SONAR_CREDENTIAL_ID = 'sonar-token'
    }

    stages {

        stage('unit test') {
            steps {
                sh 'mvn clean test'
            }
        }

        stage('sonarqube analysis') {
            steps {
                withCredentials([string(credentialsId: "$SONAR_CREDENTIAL_ID", variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('sonarqube') {
                        sh 'mvn sonar:sonar -Dsonar.projectKey=$APP_NAME'
                    }
                }
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('build & push') {
            steps {
                sh 'mvn clean package -DskipTests'
                sh 'docker build -f Dockerfile -t $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BUILD_NUMBER .'
                withCredentials([usernamePassword(passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME', credentialsId: "$DOCKER_CREDENTIAL_ID",)]) {
                    sh 'echo "$DOCKER_PASSWORD" | docker login $REGISTRY -u "$DOCKER_USERNAME" --password-stdin'
                    sh 'docker push $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BUILD_NUMBER'
                }
            }
        }

        stage('push latest') {
            when {
                branch 'master'
            }
            steps {
                sh 'docker tag $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BUILD_NUMBER $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:latest'
                sh 'docker push $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:latest'
            }
        }

        stage('deploy to dev') {
            when {
                branch 'master'
            }
            steps {
                input(id: 'deploy-to-dev', message: 'deploy to dev?')
                sh '''
                    sed -i'' "s#REGISTRY#$REGISTRY#" deploy/cicd-demo-dev.yaml
                    sed -i'' "s#DOCKERHUB_NAMESPACE#$DOCKERHUB_NAMESPACE#" deploy/cicd-demo-dev.yaml
                    sed -i'' "s#APP_NAME#$APP_NAME#" deploy/cicd-demo-dev.yaml
                    sed -i'' "s#BUILD_NUMBER#$BUILD_NUMBER#" deploy/cicd-demo-dev.yaml
                    kubectl apply -f deploy/cicd-demo-dev.yaml
                '''
            }
        }
        stage('push with tag') {
            when {
                expression {
                    return params.TAG_NAME =~ /v.*/
                }
            }
            steps {
                input(id: 'release-image-with-tag', message: 'release image with tag?')
                withCredentials([usernamePassword(credentialsId: "$GIT_CREDENTIAL_ID", passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                    sh 'git config --global user.email "liugang@wolfcode.cn" '
                    sh 'git config --global user.name "xiaoliu" '
                    sh 'git tag -a $TAG_NAME -m "$TAG_NAME" '
                    sh 'git push http://$GIT_USERNAME:$GIT_PASSWORD@$GIT_REPO_URL/$GIT_ACCOUNT/k8s-cicd-demo.git --tags --ipv4'
                }
                sh 'docker tag $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BUILD_NUMBER $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:$TAG_NAME'
                sh 'docker push $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:$TAG_NAME'
            }
        }
        stage('deploy to production') {
            when {
                expression {
                    return params.TAG_NAME =~ /v.*/
                }
            }
            steps {
                input(id: 'deploy-to-production', message: 'deploy to production?')
                sh '''
                    sed -i'' "s#REGISTRY#$REGISTRY#" deploy/cicd-demo.yaml
                    sed -i'' "s#DOCKERHUB_NAMESPACE#$DOCKERHUB_NAMESPACE#" deploy/cicd-demo.yaml
                    sed -i'' "s#APP_NAME#$APP_NAME#" deploy/cicd-demo.yaml
                    sed -i'' "s#TAG_NAME#$TAG_NAME#" deploy/cicd-demo.yaml
                    kubectl apply -f deploy/cicd-demo.yaml
                '''
            }
        }
    }
}
