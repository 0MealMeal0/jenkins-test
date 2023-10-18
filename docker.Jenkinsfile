pipeline{
    
    agent any
    
    stages{
        
        stage('git clone'){
            steps{
                git branch: 'docker', credentialsId: 'github-token', url: 'https://github.com/0MealMeal0/jenkins-test.git'
                // docker 브랜치에서 clone한다.
            }
        }
        
        stage('build'){
            steps{
                catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                // catchError: fail이 발생해도 다음 stage로 넘어가게 해준다.
                
                    sh 'docker stop nginx'
                    // container가 80포트가 할당 된 상태에서 또 할당을 하려고 하니 포트 중복 에러가 생겼다.
                    // Bind for 0.0.0.0:80 failed: port is already allocated.
                    // 그래서 이미 생성된 container를 지우고 build를 진행한다.
                    sh 'docker rmi -f $(docker images -q) '
                    // 빌드를 하게 되면 이미지들이 계속 쌓여가는 현상이 나타났다.
                    // 그래서 모든 이미지를 지운 후 build를 진행해 용량을 확보한다. 
                }
                sh 'docker build -t p1 .'        
                


            }
        }

        stage('test run') {
            steps {
                sh 'docker run -d -p 80:8080 --rm --name p1 p1'
            }
        }

        stage('application test') {
            steps {
                script {
                    String status = sh(script: "curl -sLI -w '%{http_code}' localhost:8080 -o /dev/null", returnStdout: true)
                        if (status != '200' && status != '201') {
                            error("returned status code = $status when calling test")
                        }
                }
            }
        
            post {
                failure {
                    sh 'docker stop p1'
                }
            }
        }

        stage('cleanup test') {
            steps {
                sh 'docker stop p1'
            }
        }
        
        stage('push to ecr') {
            steps {
                script {
                    sh 'aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin ${env.ecr_url}'
                    sh 'docker image tag p1:latest ${env.ecr_url}/basketball-ecr:latest'
                    sh 'docker push ${env.ecr_url}/basketball-ecr:latest'
                    
                }
            }
        }

        stage('deploy script run') {
            steps {
                script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                        sshagent (credentials: ['tf-key.']){
                            sh 'ssh -o StrictHostKeyChecking=no -i $JENKINS_HOME/tf.pem ubuntu@172.31.47.220 "docker ps -q | xargs docker stop"'
                        }
                    }
                    sshagent (credentials: ['tf-key.']){
                        sh 'ssh -o StrictHostKeyChecking=no -i $JENKINS_HOME/tf.pem ubuntu@172.31.47.220 "aws ecr get-login-password --region ap-northeast-2 | \
                        docker login --username AWS --password-stdin ${env.ecr_url}; \
                        docker run -d --rm -p  80:8080 --name nginx ${env.ecr_url}/basketball-ecr:latest"'
                        // docker run --rm 옵션으로 container stop 시 해당 container에 대한 모든 정보가 삭제된다.
                        // --rm 옵션이 없다면 container stop 시 docker ps -a 를 실행하면 stop된 container에 대한 정보가 나온다.
                        // 하지만 --rm 옵션 사용으로 container에 대한 정보가 삭제되기 때문에 docker ps -a 를 실행해도 아무런 결과가 나오지 않는다. 
                    }                          
                }
            }
        }
    }
}