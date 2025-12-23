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
            description: '版本号（留空则自动生成），一般指 BUILD_NUMBER'
        )
        choice(
            name: 'TARGET_ENV',
            choices: ['线上开发 dev', '演示 sit', '生产 prod', '海外演示 oversea-sit', '海外生产 oversea-prod'],
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
        choice(
            name: 'CLEANUP_STRATEGY',
            choices: ['none', 'full', 'smart'],
            description: '清理策略：smart=智能清理（保留缓存），full=完全清理，none=不清理'
        )
        string(
            name: 'PRE_BUILD_CONFIG_IDS',
            defaultValue: '',
            description: '编译前的配置文件 ids（多个用英文逗号分隔，留空则使用默认配置）'
        )
        string(
            name: 'POST_BUILD_CONFIG_IDS',
            defaultValue: '',
            description: '运行时的配置文件 ids（多个用英文逗号分隔，留空则使用默认配置）'
        )
        string(
            name: 'TARGET_HOSTS',
            defaultValue: '',
            description: '部署的目标服务器 target_hosts（多个用英文逗号分隔，留空则使用默认配置）'
        )
        password(
            name: 'PROD_DEPLOY_PASSWORD',
            defaultValue: '',
            description: '生产环境部署密码（仅在部署到生产环境时需要）'
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

    // environment {
    //     // 可以在这里设置环境变量

    // }

    stages {
        stage('初始化') {
            steps {
                script {
                    echo "开始执行流水线..."
                    echo "项目仓库： ${env.GIT_URL}"
                    echo "分支: ${params.BRANCH}"
                    echo "环境: ${env.TARGET_ENV}"
                    echo "版本： ${params.VERSION ?: '未配置'}"
                    echo "动作: ${params.ACTION}"
                    echo "清理策略: ${params.CLEANUP_STRATEGY}"
                    echo "使用 Docker: ${params.USE_DOCKER}"
                    echo "运行测试: ${params.RUN_TESTS}"
                    echo "编译前配置文件: ${params.PRE_BUILD_CONFIG_IDS ?: '未配置'}"
                    echo "运行时配置文件: ${params.POST_BUILD_CONFIG_IDS ?: '未配置'}"
                    echo "目标服务器: ${params.TARGET_HOSTS ?: '未配置'}"

                    // 计算实际执行的动作
                    def effectiveAction = params.ACTION

                    // 如果定义了版本号，则跳过构建直接部署
                    if (params.VERSION && params.VERSION.trim()) {
                        echo "检测到版本号已定义: ${params.VERSION}，自动切换到仅部署模式"
                        effectiveAction = 'deploy-only'
                        echo "动作已自动调整为: ${effectiveAction}"
                    }

                    // 将有效动作存储为环境变量，供后续阶段使用
                    env.EFFECTIVE_ACTION = effectiveAction

                    // 初始化共享库
                    utils()
                    env.ENVIRONMENT = getEnvName()

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
                expression { env.EFFECTIVE_ACTION.toString() == 'build-and-deploy' || env.EFFECTIVE_ACTION.toString() == 'build-only' }
            }
            steps {
                script {
                    utils.setupEnvironment()
                }
            }
        }

        stage('代码检查') {
            when {
                allOf {
                    expression { env.EFFECTIVE_ACTION.toString() == 'build-and-deploy' || env.EFFECTIVE_ACTION.toString() == 'build-only' }
                    expression { params.CHECK_CODE_STYLE.toString() == 'true' || params.CHECK_CODE_STYLE == true }
                }
            }
            steps {
                script {
                    test.runLint()
                }
            }
        }

        stage('测试') {
            when {
                allOf {
                    expression { env.EFFECTIVE_ACTION.toString() == 'build-and-deploy' || env.EFFECTIVE_ACTION.toString() == 'build-only' }
                    expression { params.RUN_TESTS.toString() == 'true' || params.RUN_TESTS == true }
                }
            }
            steps {
                script {
                    test.runTests()
                }
            }
        }

        stage('构建') {
            when {
                expression { env.EFFECTIVE_ACTION.toString() == 'build-and-deploy' || env.EFFECTIVE_ACTION.toString() == 'build-only' }
            }
            steps {
                script {
                    build.buildProject()
                }
            }
        }/

        stage('生产部署验证') {
            when {
                allOf {
                    expression { env.EFFECTIVE_ACTION.toString() == 'build-and-deploy' || env.EFFECTIVE_ACTION.toString() == 'deploy-only' }
                    expression { params.TARGET_ENV == '生产 prod' || params.TARGET_ENV == '海外生产 oversea-prod' }
                }
            }
            steps {
                script {
                    // 使用 utils 函数验证生产部署密码
                    utils.validateProdDeployPassword(params.PROD_DEPLOY_PASSWORD)

                    echo "正在部署到生产环境: ${params.TARGET_ENV}"
                }
            }
        }

        stage('部署') {
            when {
                expression { env.EFFECTIVE_ACTION.toString() == 'build-and-deploy' || env.EFFECTIVE_ACTION.toString() == 'deploy-only' }
            }
            steps {
                script {
                    deploy.deployProject()
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
