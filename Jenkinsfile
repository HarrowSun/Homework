pipeline {
    agent any

    environment {
        DEPLOY_PATH = "/var/www/libtiff"
        HOST_TRIPLET = "arm-linux-gnueabihf"
        GIT_REPO = "https://github.com/libsdl-org/libtiff"
        ARTIFACTS_DIR = "/tmp/libtiff-build-artifacts-${BUILD_NUMBER}"
    }

    stages {
        stage('Prepare Workspace') {
            steps {
                sh """
                    set -x  
                    mkdir -p ${env.ARTIFACTS_DIR}/arm ${env.ARTIFACTS_DIR}/native
                    chmod -R 777 ${env.ARTIFACTS_DIR}
                    ls -la ${env.ARTIFACTS_DIR}
                """
            }
        }

        stage('Prepare Docker Image') {
            steps {
                script {
                    try {
                        // Получаем ID пользователя Jenkins
                        def jenkins_uid = sh(script: 'id -u', returnStdout: true).trim()
                        
                        writeFile file: 'Dockerfile.build', text: """
                            FROM ubuntu:22.04
                            RUN apt-get update && \\
                                apt-get install -y \\
                                    build-essential \\
                                    gcc-${env.HOST_TRIPLET} \\
                                    g++-${env.HOST_TRIPLET} \\
                                    binutils-${env.HOST_TRIPLET} \\
                                    autoconf \\
                                    automake \\
                                    libtool \\
                                    git \\
                                    make

                            
                            RUN mkdir -p /artifacts /build && \\
                                chmod 777 /artifacts /build

                            
                            RUN useradd -u ${jenkins_uid} -m jenkins_builder && \\
                                chown -R jenkins_builder:jenkins_builder /artifacts /build

                            
                            RUN git config --system --add safe.directory '*'

                            USER jenkins_builder
                            WORKDIR /build
                        """
                        docker.build("libtiff-builder", "-f Dockerfile.build .")
                    } catch (Exception e) {
                        echo "Ошибка при подготовке Docker образа: ${e.message}"
                        throw e
                    }
                }
            }
        }

        stage('Build ARM') {
            steps {
                script {
                    try {
            
                        def containerName = "libtiff-builder-arm-${BUILD_NUMBER}"
                        
                        sh """
                            docker rm -f ${containerName} || true
                            
                            docker run --name ${containerName} -v ${env.ARTIFACTS_DIR}/arm:/artifacts libtiff-builder:latest bash -c '
                                set -x
                                id || true
                                rm -rf /build/* || true
                                ls -la /build || true
                                git clone ${env.GIT_REPO} /build
                                ls -la /build || true
                                cd /build
                                ./autogen.sh
                                ./configure \\
                                    --host=${env.HOST_TRIPLET} \\
                                    --prefix=/artifacts \\
                                    --disable-shared \\
                                    --enable-static
                                make clean
                                make -j\$(nproc)
                                make install
                                ls -la /artifacts/lib || true
                            '
                            
                            
                            docker rm -f ${containerName} || true
                        """
                    } catch (Exception e) {
                        echo "Ошибка при сборке ARM: ${e.message}"
                        throw e
                    }
                }
            }
        }

        stage('Build Native') {
            steps {
                script {
                    try {
                        
                        def containerName = "libtiff-builder-native-${BUILD_NUMBER}"
                        
                        sh """
                            
                            docker rm -f ${containerName} || true
                            
                            
                            docker run --name ${containerName} -v ${env.ARTIFACTS_DIR}/native:/artifacts libtiff-builder:latest bash -c '
                                set -x
                                id || true
                                rm -rf /build/* || true
                                ls -la /build || true
                                git clone ${env.GIT_REPO} /build
                                ls -la /build || true
                                cd /build
                                ./autogen.sh
                                ./configure \\
                                    --prefix=/artifacts \\
                                    --enable-shared \\
                                    --enable-static
                                make clean
                                make -j\$(nproc)
                                make install
                                make check || true
                                ls -la /artifacts/lib || true
                            '
                            
                            
                            docker rm -f ${containerName} || true
                        """
                    } catch (Exception e) {
                        echo "Ошибка при нативной сборке: ${e.message}"
                        throw e
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    try {
                        sh """
                            set -x  
                            
                            sudo mkdir -p ${env.DEPLOY_PATH}/bin
                            sudo mkdir -p ${env.DEPLOY_PATH}/lib
                            sudo mkdir -p ${env.DEPLOY_PATH}/share
                            sudo chown -R \$(id -u):\$(id -g) ${env.DEPLOY_PATH}
                            
                            ls -la ${env.DEPLOY_PATH}
                            
                            ls -la ${env.ARTIFACTS_DIR}/arm/lib || true
                            ls -la ${env.ARTIFACTS_DIR}/native/lib || true
                            
                            cp ${env.ARTIFACTS_DIR}/arm/lib/libtiff.a ${env.DEPLOY_PATH}/lib/ || echo "No ARM static lib found"
                            cp -r ${env.ARTIFACTS_DIR}/native/bin/* ${env.DEPLOY_PATH}/bin/ || echo "No native binaries found"
                            cp -r ${env.ARTIFACTS_DIR}/native/lib/* ${env.DEPLOY_PATH}/lib/ || echo "No native libs found"
                            cp -r ${env.ARTIFACTS_DIR}/native/share/* ${env.DEPLOY_PATH}/share/ || echo "No shared files found"
                            
                            ls -la ${env.DEPLOY_PATH}/lib
                            
                            cd ${env.DEPLOY_PATH}/lib
                            if [ -f libtiff.so.6.1.0 ]; then
                                ln -sf libtiff.so.6.1.0 libtiff.so
                                ln -sf libtiff.so.6.1.0 libtiff.so.6
                            elif [ -f libtiff.so.* ]; then
                                SONAME=\$(ls libtiff.so.* | head -1)
                                ln -sf \$SONAME libtiff.so
                                ln -sf \$SONAME \$(echo \$SONAME | cut -d. -f1-2)
                            fi
                        """
                    } catch (Exception e) {
                        echo "Ошибка при деплое: ${e.message}"
                        throw e
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'docker rmi -f libtiff-builder || true'
            sh "rm -rf ${env.ARTIFACTS_DIR} Dockerfile.build || true"
        }
        success {
            echo "Сборка успешно завершена!"
        }
        failure {
            echo "Сборка завершилась с ошибкой"
        }
    }
}
