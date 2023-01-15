# jenkins

[TOC]

## 部署

docker部署：

```shell
docker run -d --restart=always \
-p 8081:8080 \
-p 50000:50000 \
-v /etc/localtime:/etc/localtime:ro \
-v /etc/timezone:/etc/timezone:ro \
-v /home/wzg/jenkinsci:/var/jenkins_home \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /home/wzg:/home \
--privileged=true \
--name jenkins \
jenkinsci/blueocean
```

因为jenkins流水线会有部署容器的操作，即需要**docker-in-docker**，两种方式：

1. 使用特权`--privileged=true`；

2. 挂在`docker.sock`：`-v /var/run/docker.sock:/var/run/docker.sock`

另外，需要和宿主机的时间一致：

- 挂载时间：`-v /etc/localtime:/etc/localtime:ro` 
- 挂载时区：`-v /etc/timezone:/etc/timezone:ro`

参考：https://shisho.dev/blog/posts/docker-in-docker/



## 配置

### gitlab

### maven

### 邮件通知

### 一些插件

```
Config File Provider配置文件
PMD 静态代码分析器
JUNIT单元测试
JACOCO 测试覆盖率
Performance：使用Taurus进行性能测试、
Credential binding plugin插件
HashiCorp Vault插件管理凭证
Nexus 制品管理
```



## 流水线

Jenkinsfile参考例子：

```groovy
pipeline {
    agent {
        label 'agent'
    }
    environment {
        MONITOR_COMMON="monitor-common"
        MONITOR_CONSUMER="monitor-consumer"
        MONITOR_PROVIDER="monitor-provider"
        DOCKER_REPOSITORY="colorlightwzg"
        CCLOUD_MONITOR_CONSUMER="ccloud-monitor-consumer"
        CCLOUD_MONITOR_PROVIDER="ccloud-monitor-provider"
    }
    options {
        timeout(time: 1, unit: 'HOURS')
    }
    stages {
        stage('Prepare') {
            steps {
                echo "当前工作空间=${WORKSPACE}"
                sh 'printenv'
            }
        }
        stage('Unit Test') {
            agent {
                docker {
                    image 'maven:3.8.6-jdk-11'
                    args '-v /root/.m2:/root/.m2 -v /media/colorlight/sdb1/project_ccloud/jenkinsci/my_config/maven:/usr/share/maven/ref'
                }
            }
            steps {
                echo "当前工作空间=${WORKSPACE}"
                jacoco()
                sh 'mvn clean test -Ptest'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                    jacoco(
                        execPattern: '**/target/**/*.exec',
                        classPattern: '**/target/classes',
                        exclusionPattern: '**/src/test*',
                        inclusionPattern: '**/src/main/java'
                    )
                    sh 'mvn jacoco:report'
                }
            }
        }
        stage("Build") {
            agent {
                docker {
                    image 'maven:3.8.6-jdk-11'
                    args '-v /root/.m2:/root/.m2 -v /media/colorlight/sdb1/project_ccloud/jenkinsci/my_config/maven:/usr/share/maven/ref'
                }
            }
            steps {
                echo "当前工作空间=${WORKSPACE}"
                sh 'mvn clean package -Pdocker -Dmaven.test.skip=true'
            }
            post {
                 unstable {
                     archiveArtifacts artifacts: "**/target/*.jar", fingerprint: true, onlyIfSuccessful: true
                     stash includes: '**/target/*.jar', name: 'JARS'
                 }
                success {
                    archiveArtifacts artifacts: "**/target/*.jar", fingerprint: true, onlyIfSuccessful: true
                    stash includes: '**/target/*.jar', name: 'JARS'
                }
            }
        }
        stage("Push Nexus Artifact"){
            parallel {
                stage("monitor-parent") {
                    steps {
                        echo "当前工作空间=${WORKSPACE}"
                        unstash 'JARS'
                        script {
                            def pom = readMavenPom file: "pom.xml"
                            def repository=getRepository(pom.parent.version)
                            nexusArtifactUploader artifacts: [[artifactId: "${pom.artifactId}",
                                                                classifier: '',
                                                                file: "pom.xml",
                                                                type: "pom"]],
                                                  credentialsId: 'Nexus-192-168-1-83-8082-CREDENTIAL',
                                                  groupId: "${pom.groupId}",
                                                  nexusUrl: '192.168.1.83:8082',
                                                  nexusVersion: 'nexus3',
                                                  protocol: 'http',
                                                  repository: "${repository}",
                                                  version: "${pom.version}"
                        }
                    }
                }
                stage("monitor-common") {
                    steps {
                        echo "当前工作空间=${WORKSPACE}"
                        unstash 'JARS'
                        script {
                            def pom = readMavenPom file: "${MONITOR_COMMON}/pom.xml"
                            def repository=getRepository(pom.parent.version)
                            nexusArtifactUploader artifacts: [[artifactId: "${pom.artifactId}",
                                                                classifier: '',
                                                                file: "./${MONITOR_COMMON}/target/${pom.artifactId}-${pom.parent.version}.${pom.packaging}",
                                                                type: "${pom.packaging}"],
                                                              /* [artifactId: "${pom.artifactId}",
                                                                classifier: '',
                                                                file: "./${MONITOR_COMMON}/target/${pom.artifactId}-${pom.parent.version}-sources.${pom.packaging}",
                                                                type: "${pom.packaging}"], */
                                                              [artifactId: "${pom.artifactId}",
                                                                classifier: '',
                                                                file: "${MONITOR_COMMON}/pom.xml",
                                                                type: "pom"]],
                                                  credentialsId: 'Nexus-192-168-1-83-8082-CREDENTIAL',
                                                  groupId: "${pom.parent.groupId}",
                                                  nexusUrl: '192.168.1.83:8082',
                                                  nexusVersion: 'nexus3',
                                                  protocol: 'http',
                                                  repository: "${repository}",
                                                  version: "${pom.parent.version}"
                        }//finish script
                    }
                }
                stage("monitor-provider") {
                    steps {
                        echo "当前工作空间=${WORKSPACE}"
                        unstash 'JARS'
                        script {
                            def pom = readMavenPom file: "${MONITOR_PROVIDER}/pom.xml"
                            def repository=getRepository(pom.parent.version)
                            nexusArtifactUploader artifacts: [[artifactId: "${pom.artifactId}",
                                                                classifier: '',
                                                                file: "./${MONITOR_PROVIDER}/target/${pom.artifactId}-${pom.parent.version}.${pom.packaging}",
                                                                type: "${pom.packaging}"],
                                                              /* [artifactId: "${pom.artifactId}",
                                                                classifier: '',
                                                                file: "./${MONITOR_PROVIDER}/target/${pom.artifactId}-${pom.parent.version}-sources.${pom.packaging}",
                                                                type: "${pom.packaging}"], */
                                                              [artifactId: "${pom.artifactId}",
                                                                classifier: '',
                                                                file: "${MONITOR_PROVIDER}/pom.xml",
                                                                type: "pom"]],
                                                  credentialsId: 'Nexus-192-168-1-83-8082-CREDENTIAL',
                                                  groupId: "${pom.parent.groupId}",
                                                  nexusUrl: '192.168.1.83:8082',
                                                  nexusVersion: 'nexus3',
                                                  protocol: 'http',
                                                  repository: "${repository}",
                                                  version: "${pom.parent.version}"
                        }//finish script
                    }
                }
                stage("monitor-consumer") {
                    steps {
                        echo "当前工作空间=${WORKSPACE}"
                        unstash 'JARS'
                        script {
                            def pom = readMavenPom file: "${MONITOR_CONSUMER}/pom.xml"
                            def repository=getRepository(pom.parent.version)
                            nexusArtifactUploader artifacts: [[artifactId: "${pom.artifactId}",
                                                                classifier: '',
                                                                file: "./${MONITOR_CONSUMER}/target/${pom.artifactId}-${pom.parent.version}.${pom.packaging}",
                                                                type: "${pom.packaging}"],
                                                              /* [artifactId: "${pom.artifactId}",
                                                                classifier: '',
                                                                file: "./${MONITOR_CONSUMER}/target/${pom.artifactId}-${pom.parent.version}-sources.${pom.packaging}",
                                                                type: "${pom.packaging}"], */
                                                              [artifactId: "${pom.artifactId}",
                                                                classifier: '',
                                                                file: "${MONITOR_CONSUMER}/pom.xml",
                                                                type: "pom"]],
                                                  credentialsId: 'Nexus-192-168-1-83-8082-CREDENTIAL',
                                                  groupId: "${pom.parent.groupId}",
                                                  nexusUrl: '192.168.1.83:8082',
                                                  nexusVersion: 'nexus3',
                                                  protocol: 'http',
                                                  repository: "${repository}",
                                                  version: "${pom.parent.version}"
                        }//finish script
                    }
                }
            }
        }
        stage("Docker Build & Push: Registry") {
            parallel {
                stage("ccloud-monitor-provider") {
                    steps {
                        echo "当前工作空间=${WORKSPACE}"
                        unstash 'JARS'
                        script {
                            def pom = readMavenPom file: "${MONITOR_PROVIDER}/pom.xml"
                            def _image_name = getImageName("${DOCKER_REPOSITORY}/${CCLOUD_MONITOR_PROVIDER}", pom.parent.version)
                            def _jar_name = "target/${pom.artifactId}-${pom.parent.version}.${pom.packaging}"
                            def customImage = docker.build("${_image_name}", "--build-arg artifact=${MONITOR_PROVIDER}/${_jar_name} -f ./Dockerfile .")
                            customImage.push()
                        }
                    }
                    post {
                        failure {
                            echo "push images:${_image_name} fail"
                        }
                    }
                }
                stage("ccloud-monitor-consumer") {
                    steps {
                        echo "当前工作空间=${WORKSPACE}"
                        unstash 'JARS'
                        script {
                            def pom = readMavenPom file: "${MONITOR_CONSUMER}/pom.xml"
                            def _image_name = getImageName("${DOCKER_REPOSITORY}/${CCLOUD_MONITOR_CONSUMER}", pom.parent.version)
                            def _jar_name = "target/${pom.artifactId}-${pom.parent.version}.${pom.packaging}"
                            def customImage = docker.build("${_image_name}", "--build-arg artifact=${MONITOR_CONSUMER}/${_jar_name} -f ./Dockerfile .")
                            customImage.push()
                        }
                    }
                    post {
                        failure {
                            echo "push images:${_image_name} fail"
                        }
                    }
                }
            }
            post {
                success {
                    cleanWs()
                }
            }
        }
    }
    post {
        always{
            cleanWs()
            script{
                emailext attachLog: true, body: '''${SCRIPT, template="groovy-html.template"}''',
                    compressLog: true,
                    mimeType: 'text/html',
                    recipientProviders: [upstreamDevelopers(), developers(), requestor(), buildUser(), brokenTestsSuspects(), brokenBuildSuspects(), culprits()],
                    replyTo: '2995519898@qq.com',
                    subject: "Jenkins构建结果报告:${currentBuild.result?:'SUCCESS'} - '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                    to: '2156743750@qq.com 1939486642@qq.com 909753735@qq.com 639590381@qq.com 1545758198@qq.com 1295965985@qq.com'
            }
        }
    }
}
def getRepository(String version) {
    String suffix = "snapshots";
    String[] strings = version.split('-');
    if (strings.length > 0 && strings[strings.length - 1].toLowerCase() == "release") {
        suffix = "releases";
    }
    return "maven-" + suffix;
}
def getImageName(String repository, String tag) {
    return repository + ":" + tag + "-" + new Date().format('yyMMdd') + ".${env.BUILD_NUMBER}";
}
```



