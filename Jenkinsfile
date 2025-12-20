@Library('multi-language-pipeline') _

pipeline {
    agent any

    parameters {
        gitParameter(
            branchFilter: 'origin/.*',
            tagFilter: 'v.*',
            defaultValue: 'main',
            name: 'BRANCH',
            type: 'PT_BRANCH_TAG',
            selectedValue: 'DEFAULT',
            sortMode: 'DESCENDING_SMART',
            quickFilterEnabled: true,
            description: '选择要构建的分支或标签'
        )
        string(
            name: 'VERSION',
            defaultValue: '',
            description: '版本号（留空则从项目文件自动读取）'
        )
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'sit', 'prod', 'oversea-sit', 'oversea-prod'],
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
        choice(
            name: 'CLEANUP_STRATEGY',
            choices: ['none', 'full', 'smart'],
            description: '清理策略：smart=智能清理（保留缓存），full=完全清理，none=不清理'
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
    }

    stages {
        stage('初始化') {
            steps {
                script {
                    env.LANGUAGE = utils.checkLanguage()
                    echo "开始执行流水线..."
                    echo "项目仓库： ${env.GIT_URL}"
                    echo "分支: ${params.BRANCH}"
                    echo "环境: ${params.ENVIRONMENT}"
                    echo "语言：${env.LANGUAGE}"
                    echo "动作: ${params.ACTION}"
                    echo "清理策略: ${params.CLEANUP_STRATEGY}"
                    echo "使用 Docker: ${params.USE_DOCKER}"
                    echo "运行测试: ${params.RUN_TESTS}"
                    echo "在 Web App 中构建: ${params.BUILD_IN_WEB_APP}"

                    // 初始化共享库
                    utils()

                    // 使用dir步骤确保工作目录不变
                    dir('shared-lib') {
                        git(
                            url: 'git@github.com:choice-form/jenkins-shared-library.git',
                            branch: 'main',
                            credentialsId: 'github_ssh'
                        )
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
                    utils.setupEnvironment(env.LANGUAGE)
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
                    test.runLint(env.LANGUAGE)
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
                    test.runTests(env.LANGUAGE)
                }
            }
        }

        stage('构建') {
            when {
                expression { params.ACTION.toString() == 'build-and-deploy' || params.ACTION.toString() == 'build-only' }
            }
            steps {
                script {
                    build.buildProject()
                }
            }
        }

        stage('部署') {
            when {
                expression { params.ACTION.toString() == 'build-and-deploy' || params.ACTION.toString() == 'deploy-only' }
            }
            steps {
                script {
                    deploy.deployProject(env.LANGUAGE, params.ENVIRONMENT)
                }
            }
        }
    }

    post {
        always {
            script {
                echo "流水线执行完成"
                utils.cleanWorkspace()
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
