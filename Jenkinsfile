pipeline {
    agent any

    environment {
        HARBOR_ADDR = '192.168.31.119:80'
        HARBOR_REPO = 'repo'
        IMAGE_NAME  = "${JOB_NAME}".toLowerCase()
        CONTAINER_NAME = "${JOB_NAME}".toLowerCase()
        FULL_IMAGE  = "${HARBOR_ADDR}/${HARBOR_REPO}/${IMAGE_NAME}:${Tag}"
    }

    tools {
        maven 'Maven_3.9.12'
    }

    stages {
        stage('git拉取代码') {
            steps {
                checkout scmGit(branches: [[name: "${Tag}"]], extensions: [], userRemoteConfigs: [[url: 'http://192.168.31.119:8081/root/java.git']])
            }
        }

        stage('maven项目构建 & sonarqube代码检测') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh "mvn clean package sonar:sonar -DskipTests -Dsonar.projectName=${JOB_NAME} -Dsonar.projectKey=${JOB_NAME}"
                }
            }
        }

        stage('docker制作镜像') {
            steps {
                sh """
                    cp ./target/*.jar ./docker/
                    docker build -t ${IMAGE_NAME}:${Tag} ./docker/
                """
            }
        }

        stage('harbor镜像推送') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'Harbor_Account', usernameVariable: 'HB_USER', passwordVariable: 'HB_PASS')]) {
                    sh """
                        echo "${HB_PASS}" | docker login -u "${HB_USER}" --password-stdin ${HARBOR_ADDR}
                        docker tag ${IMAGE_NAME}:${Tag} ${FULL_IMAGE}
                        docker push ${FULL_IMAGE}
                    """
                }
            }
        }

        stage('docker发布应用') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'Harbor_Account', usernameVariable: 'HB_USER', passwordVariable: 'HB_PASS')]) {
                    sshPublisher(publishers: [
                        sshPublisherDesc(
                            configName: 'docker_server',
                            verbose: true,
                            transfers: [
                                sshTransfer(
                                    cleanRemote: false,
                                    execCommand: """
                                        # 1. 登录 Harbor
                                        echo "${HB_PASS}" | docker login -u "${HB_USER}" --password-stdin ${HARBOR_ADDR}

                                        # 2. 拉取新镜像
                                        docker pull ${FULL_IMAGE}

                                        # 3. 判断并清理旧容器
                                        if [ \$(docker ps -aq -f name=^/${CONTAINER_NAME}\$) ]; then
                                            docker rm -f ${CONTAINER_NAME}
                                        fi

                                        # 4. 启动新容器
                                        docker run -d \\
                                            --name ${CONTAINER_NAME} \\
                                            -p ${env.HostPort ?: '8083'}:${env.ContainerPort ?: '8080'} \\
                                            --restart always \\
                                            ${FULL_IMAGE}

                                        # 5. 清理无用镜像
                                        docker image prune -f
                                    """.stripIndent(),
                                    execTimeout: 120000
                                )
                            ]
                        )
                    ])
                }
            }
        }
    }
}
