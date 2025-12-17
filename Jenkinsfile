@Library('multi-language-pipeline') _

pipeline {
    agent any

    parameters {
        choice(
            name: 'LANGUAGE',
            choices: ['elixir', 'node', 'go', 'python'],
            description: '选择项目语言'
        )
        choice(
            name: 'ENVIRONMENT',
            choices: ['sit', 'dev', 'prod', 'oversea-sit', 'oversea-prod'],
            description: '选择部署环境'
        )
        booleanParam(
            name: 'USE_DOCKER',
            defaultValue: false,
            description: '是否使用 Docker 部署'
        )
        booleanParam(
            name: 'RUN_TESTS',
            defaultValue: true,
            description: '是否运行测试'
        )
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    environment {
        // 可以在这里设置环境变量
        ELIXIR_VERSION = '1.19.4-otp-28'
        ERLANG_VERSION = '28.1.1'
    }

    stages {
        stage('初始化') {
            steps {
                script {
                    echo "开始执行流水线..."
                    echo "语言: ${params.LANGUAGE}"
                    echo "环境: ${params.ENVIRONMENT}"
                    echo "使用 Docker: ${params.USE_DOCKER}"

                    // 初始化共享库
                    utils()
                }
            }
        }

        stage('环境设置') {
            steps {
                script {
                    utils.setupEnvironment(params.LANGUAGE)
                }
            }
        }

        stage('代码检查') {
            steps {
                script {
                    test.runLint(params.LANGUAGE)
                }
            }
        }

        stage('构建') {
            steps {
                script {
                    build.buildProject(params.LANGUAGE)
                }
            }
        }

        stage('测试') {
            when {
                expression { params.RUN_TESTS == 'true' }
            }
            steps {
                script {
                    test.runTests(params.LANGUAGE)
                }
            }
        }

        stage('部署') {
            when {
                expression { params.ENVIRONMENT != 'dev' }
            }
            steps {
                script {
                    deploy.deployProject(params.LANGUAGE, params.ENVIRONMENT)
                }
            }
        }
    }

    post {
        always {
            script {
                echo "流水线执行完成"
                cleanWs()
            }
        }
        success {
            script {
                echo "流水线执行成功!"
                utils.sendNotification('SUCCESS')
            }
        }
        failure {
            script {
                echo "流水线执行失败!"
                utils.sendNotification('FAILURE')
            }
        }
    }
}
