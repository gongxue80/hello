@Library('multi-language-pipeline') _

pipeline {
    agent any

    parameters {
        gitParameter(
            branchFilter: 'origin/.*',
            defaultValue: 'main',
            name: 'BRANCH',
            type: 'PT_BRANCH_TAG',
            selectedValue: 'DEFAULT',
            sortMode: 'DESCENDING_SMART',
            description: '选择要构建的分支或标签'
        )
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
        choice(
            name: 'ACTION',
            choices: ['build-and-deploy', 'build-only', 'deploy-only'],
            description: '执行动作'
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
        booleanParam(
            name: 'CHECK_CODE_STYLE',
            defaultValue: true,
            description: '是否检查代码风格'
        )
        booleanParam(
            name: 'BUILD_IN_WEB_APP',
            defaultValue: false,
            description: '是否在 Web App 中构建（仅适用于 Elixir）'
        )
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    triggers {
        pollSCM('H/5 * * * *')
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
                    echo "项目仓库： ${env.GIT_URL}"
                    echo "分支: ${params.BRANCH}"
                    echo "语言: ${params.LANGUAGE}"
                    echo "环境: ${params.ENVIRONMENT}"
                    echo "使用 Docker: ${params.USE_DOCKER}"
                    echo "运行测试: ${params.RUN_TESTS}"
                    echo "在 Web App 中构建: ${params.BUILD_IN_WEB_APP}"

                    // 初始化共享库
                    utils()
                    dir('project') {
                        checkout scmGit(branches: [[name: params.BRANCH]],
                            userRemoteConfigs: [[url: env.GIT_URL]])
                    }
                }
            }
        }

        stage('环境设置') {
            when {
                expression { params.ACTION.toString() == 'build-and-deploy' || params.ACTION.toString() == 'build-only' }
            }
            steps {
                script {
                    utils.setupEnvironment(params.LANGUAGE)
                }
            }
        }

        stage('代码检查') {
            when {
                allOf {
                    expression { params.ACTION.toString() == 'build-and-deploy' || params.ACTION.toString() == 'build-only' }
                    expression { params.CHECK_CODE_STYLE.toString() == 'true' || params.CHECK_CODE_STYLE == true }
                }
            }
            steps {
                script {
                    test.runLint(params.LANGUAGE)
                }
            }
        }

        stage('测试') {
            when {
                allOf {
                    expression { params.ACTION.toString() == 'build-and-deploy' || params.ACTION.toString() == 'build-only' }
                    expression { params.RUN_TESTS.toString() == 'true' || params.RUN_TESTS == true }
                }
            }
            steps {
                script {
                    test.runTests(params.LANGUAGE)
                }
            }
        }

        stage('构建') {
            when {
                expression { params.ACTION.toString() == 'build-and-deploy' || params.ACTION.toString() == 'build-only' }
            }
            steps {
                script {
                    build.buildProject(params.LANGUAGE)
                }
            }
        }

        stage('部署') {
            when {
                expression { params.ACTION.toString() == 'build-and-deploy' || params.ACTION.toString() == 'deploy-only' }
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
