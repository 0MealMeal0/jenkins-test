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
                    sh 'docker rmi -f $(docker images -q)'
                    // 빌드를 하게 되면 이미지들이 계속 쌓여가는 현상이 나타났다.
                    // 그래서 모든 이미지를 지운 후 build를 진행해 용량을 확보한다. 
                }
                sh 'docker build -t p1 .'        
            }
        }

        stage('test run') {
            steps {
                sh 'docker run -d -p 8080:80 --rm --name p1 p1'
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
                    sh "aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin ${env.ECR_URL}"
                    sh "docker image tag p1:latest ${env.ECR_URL}/basketball-ecr:latest"
                    sh "docker push ${env.ECR_URL}/basketball-ecr:latest"
                }
            }
        }

        stage('deploy script run') {
            steps {
                script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                        sshagent (credentials: ['tf-key.']){
                            sh 'ssh -o StrictHostKeyChecking=no -i $JENKINS_HOME/tf.pem ubuntu@172.31.27.102 "docker ps -q | xargs docker stop"'
                        }
                    }
                    sshagent (credentials: ['tf-key.']){
                        // sh 'ssh -o StrictHostKeyChecking=no -i $JENKINS_HOME/tf.pem ubuntu@172.31.27.102  \
                        //    "docker login --username AWS --password-stdin 175321200489.dkr.ecr.ap-northeast-2.amazonaws.com; \
                        //     docker run -d --rm -p 80:80 --name nginx 175321200489.dkr.ecr.ap-northeast-2.amazonaws.com/basketball-ecr:latest"'
                        
                        sh "ssh -o StrictHostKeyChecking=no -i $JENKINS_HOME/tf.pem ubuntu@172.31.27.102 | \
                            aws ecr get-login-password --region ap-northeast-2 | \
                            docker login --username AWS --password-stdin ${env.ECR_URL}; \
                            docker run -d --rm -p  8080:80 --name nginx ${env.ECR_URL}/basketball-ecr:latest"
                        // docker run --rm 옵션으로 container stop 시 해당 container에 대한 모든 정보가 삭제된다.
                        // --rm 옵션이 없다면 container stop 시 docker ps -a 를 실행하면 stop된 container에 대한 정보가 나온다.
                        // 하지만 --rm 옵션 사용으로 container에 대한 정보가 삭제되기 때문에 docker ps -a 를 실행해도 아무런 결과가 나오지 않는다. 
                    }                          
                }
            }
        }
    }
}

//  try {sh 'sshpass -p 1234 ssh ubuntu@172.31.44.9 -o StrictHostKeyChecking=no "docker ps -a -q | xargs docker rm -f"'} 
//                             catch (Exception e) {// 스크립트가 실패해도 계속 진행
//                             echo "Failed to stop containers: ${e.getMessage()}"
//                             }
//                     sh 'sshpass -p 1234 ssh ubuntu@172.31.44.9 -o StrictHostKeyChecking=no  \
//                         "docker pull public.ecr.aws/abctest/btcms:latest;docker run -d --rm -p 80:80 --name nginx public.ecr.aws/abctest/btcms:latest;curl 127.0.0.1"'
