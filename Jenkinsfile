// java项目

// 需要解析http返回结果时使用
import groovy.json.JsonSlurperClassic
import groovy.json.JsonOutput

// 按需求更改变量的值
node {
    // 参数设置
    def repoUrl = 'git@*****.git'
    def repoBranch = 'dev'
    def gitDir = ''
    def workspace = ''

    // docker镜像信息
    def imageName = "*****/demo"
    def imageTag = "dev"

    // rancher1.X部署所需变量
    def rancher1Url = 'https://<rancher1_service>/v2-beta'  // rancher1.X地址 
    def rancher1Env = ''  // rancher1.X需部署服务所在环境的ID
    def rancher1Service = '<stack>/<service>'   // rancher1.X需部署服务的 栈/服务名

    // rancher2.X部署所需变量
    def rancher2Url = "https://<rancher2_service>/v3/project/<cluster_id>:<project_id>" // rancher2.X地址+project
    def rancher2Namespace = ""  // rancher2.X需部署服务的命名空间
    def rancher2Service = "bookkeeper"  // rancher2.X需部署服务的服务名

    def recipients = ''    // 收件人
    def jenkinsHost = ''    // jenkins服务器地址

    // 认证ID
    def gitCredentialsId = 'aa651463-c335-4488-8ff0-b82b87e11c59'
    def settingsConfigId = '3ae4512e-8380-4044-9039-2b60631210fe'
    def rancherAPIKey = 'd41150be-4032-4a53-be12-3024c6eb4204'
    def soundsId = '69c344f1-8b11-47a1-a3b6-dfa423b94d78'

    // 工具配置
    def mvnHome = 'maven3.5.2'

    env.SONAR_HOME = "${tool 'sonarscanner'}"
    env.PATH="${env.SONAR_HOME}/bin:${env.PATH}"

    try {
        // 正常构建流程
        stage("Preparation"){
            // 拉取需要构建的代码
            git_maps = checkout scm: [$class: 'GitSCM', branches: [[name: "*/${repoBranch}"]], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: "${gitDir}"]], userRemoteConfigs: [[credentialsId: "${gitCredentialsId}", url: "${repoUrl}"]]]
            env.GIT_REVISION = git_maps.GIT_COMMIT[0..8]
        }
        try {
            stage('BackEndBuild') {
                // maven构建生成二进制包
                dir("${workspace}"){
                    withMaven(
                        maven: "${mvnHome}", 
                        mavenSettingsConfig: "${settingsConfigId}",
                        options: [artifactsPublisher(disabled: true)]) {
                        sh "mvn -U clean package -Dmaven.test.skip=true"
                    }
                }
            }
        }finally {
            stage('ArchiveJar') {
                // 获取二进制包产物
                archiveArtifacts allowEmptyArchive: true, artifacts: "${workspace}target/surefire-reports/TEST-*.xml"
                archiveArtifacts allowEmptyArchive: true, artifacts: "${workspace}target/*.jar"
            }
        }
        stage('CodeCheck'){
            // sonar_scanner进行代码静态检查
            withSonarQubeEnv('sonar') {
                sh """
                sonar-scanner -X -Dsonar.language=java \
                -Dsonar.projectKey=$JOB_NAME \
                -Dsonar.projectName=$JOB_NAME \
                -Dsonar.projectVersion=$GIT_REVISION \
                -Dsonar.sources=src/ \
                -Dsonar.sourceEncoding=UTF-8 \
                -Dsonar.java.binaries=target/ \
                -Dsonar.exclusions=src/test/**
                """
            }
        }
        stage("QualityGate") {
            // 获取sonar检查结果
            timeout(time: 1, unit: "HOURS") {
                def qg = waitForQualityGate()
                if (qg.status != 'OK') {
                    error "Pipeline aborted due to quality gate failure: ${qg.status}"
                }
            }
        }
        stage('DockerBuild'){
            // docker生成镜像并push到远程仓库中 
            sh """
                rm -f ${workspace}src/docker/*.jar
                cp ${workspace}target/*.jar ${workspace}src/docker/
            """
            dir("${workspace}src/docker/"){
                def image = docker.build("${imageName}:${imageTag}")
                image.push()
                sh "docker rmi ${imageName}:${imageTag}"
            }
        }
        stage('Rancher1Deploy'){
            // Rancher1.X上进行服务的更新部署
            rancher confirm: false, credentialId: "${rancherAPIKey}", endpoint: "${rancher1Url}", environmentId: "${rancher1Env}", environments: '', image: "${imageName}:${imageTag}", ports: '', service: "${rancher1Service}"
        }
        stage('Rancher2Deploy') {
            // Rancher2.X上更新容器
            // 获取服务信息
            def response = httpRequest acceptType: 'APPLICATION_JSON', authentication: "${rancherAPIKey}", contentType: 'APPLICATION_JSON', httpMode: 'GET', responseHandle: 'LEAVE_OPEN', timeout: 10, url: "${rancher2Url}/workloads/deployment:${rancher2Namespace}:${rancher2Service}"
            def serviceInfo = new JsonSlurperClassic().parseText(response.content)
            response.close()
            
            def dockerImage = imageName+":"+imageTag
            if (dockerImage.equals(serviceInfo.containers[0].image)) {
                // 如果镜像名未改变，直接删除原容器
                // 查询容器名称
                response = httpRequest acceptType: 'APPLICATION_JSON', authentication: "${rancherAPIKey}", contentType: 'APPLICATION_JSON', httpMode: 'GET', responseHandle: 'LEAVE_OPEN', timeout: 10, url: "${rancher2Url}/pods/?workloadId=deployment:${rancher2Namespace}:${rancher2Service}"
                def podsInfo = new JsonSlurperClassic().parseText(response.content)
                def containerName = podsInfo.data[0].name
                response.close()
                // 删除容器
                httpRequest acceptType: 'APPLICATION_JSON', authentication: "${rancherAPIKey}", contentType: 'APPLICATION_JSON', httpMode: 'DELETE', responseHandle: 'NONE', timeout: 10, url: "${rancher2Url}/pods/${rancher2Namespace}:${containerName}"
                
            } else {
                // 如果镜像名改变，使用新镜像名更新服务
                serviceInfo.containers[0].image = dockerImage
                def updateJson = new JsonOutput().toJson(serviceInfo)
                httpRequest acceptType: 'APPLICATION_JSON', authentication: "${rancherAPIKey}", contentType: 'APPLICATION_JSON', httpMode: 'PUT', requestBody: "${updateJson}", responseHandle: 'NONE', timeout: 10, url: "${rancher2Url}/workloads/deployment:${rancher2Namespace}:${rancher2Service}"
            }
        }

        // 构建成功：声音提示，邮件发送
        httpRequest authentication:"${soundsId}", url:'http://${jenkinsHost}/sounds/playSound?src=file:///var/jenkins_home/sounds/4579.wav'
        emailext attachlog:true, body: '$DEFAULT_CONTENT', subject: '$DEFAULT_SUBJECT', to: "${recipients}"
    } catch(err) {
        // 构建失败：声音提示，邮件发送
        currentBuild.result = 'FAILURE'
        httpRequest authentication:"${soundsId}", url:'http://${jenkinsHost}/sounds/playSound?src=file:///var/jenkins_home/sounds/8923.wav'
        emailext attachlog:true, body: '$DEFAULT_CONTENT', subject: '$DEFAULT_SUBJECT', to: "${recipients}"
    }
}
