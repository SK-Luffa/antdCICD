pipeline {
    agent any

    environment {
        // ============== 根据你的项目修改这里 ==============
        APP_NAME        = 'react-frontend'
        EXPOSE_PORT     = '8000'          // 外部访问端口
        INTERNAL_PORT   = '80'             // 容器内 Nginx 端口
        IMAGE_NAME      = "react-frontend:${BUILD_NUMBER}"
        // =================================================
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))  // 保留最近10次构建
        timeout(time: 20, unit: 'MINUTES')
        disableConcurrentBuilds()  // 禁止并发构建
    }

    stages {
        stage('📌 环境检查') {
            steps {
                echo "===== 打印环境信息 ====="
                sh 'node -v'
                sh 'pnpm -v'
                sh 'docker -v'
                sh 'git --version'
            }
        }

        stage('📥 拉取代码') {
            steps {
                echo "===== 从 GitHub 拉取代码 ====="
                checkout scm
                sh 'ls -la'
            }
        }

        stage('📦 安装依赖') {
            steps {
                echo "===== pnpm install（使用淘宝镜像加速）====="
                sh '''
                    pnpm config set registry https://registry.npmmirror.com
                    pnpm install --prefer-offline
                '''
            }
        }

        stage('🔨 构建产物') {
            steps {
                echo "===== pnpm run build ====="
                sh 'pnpm run build'
                echo "构建完成，产物大小："
                sh 'du -sh dist'
            }
        }

        stage('🐳 构建 Docker 镜像') {
            steps {
                echo "===== 构建镜像：${IMAGE_NAME} ====="
                sh '''
                    docker build -t ${IMAGE_NAME} .
                    docker images | head -5
                '''
            }
        }

        stage('🚀 部署到服务器') {
            steps {
                echo "===== 停止旧容器 & 启动新容器 ====="
                sh '''
                    # 停止并删除同名旧容器（忽略错误）
                    docker stop ${APP_NAME} || true
                    docker rm ${APP_NAME} || true

                    # 启动新容器（主机端口 8000 -> 容器 80
                    docker run -d \\
                      --name ${APP_NAME} \\
                      --restart=always \\
                      -p ${EXPOSE_PORT}:${INTERNAL_PORT} \\
                      -v /etc/localtime:/etc/localtime:ro \\
                      ${IMAGE_NAME}

                    # 显示运行中的容器
                    echo "容器启动情况："
                    docker ps --filter name=${APP_NAME}

                    # 清理旧镜像（保留最近 5 个）
                    docker images -q react-frontend 2>/dev/null | tail -n +6 | xargs -r docker rmi -f || true
                '''
            }
        }

        stage('🧪 健康检查') {
            steps {
                echo "===== 验证部署是否成功 ====="
                script {
                    sleep 5
                    def status = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8000", returnStdout: true).trim()
                    echo "HTTP 状态码：${status}"
                    if (status == '200') {
                        echo "✅ 部署成功！访问地址：http://121.36.244.239:8000"
                    } else {
                        echo "⚠️ 状态码异常，请检查容器日志："
                        sh 'docker logs --tail 30 react-frontend || true'
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
    }

    post {
        success {
            echo "🎉🎉🎉 CI/CD 流程全部完成！构建号：${BUILD_NUMBER}"
        }
        failure {
            echo "❌ 构建失败！请查看日志排查"
            sh 'docker ps -a || true'
        }
        always {
            echo "===== 本次流水线结束 ====="
        }
    }
}
