def getValue(Map map, String key, def defaultValue) {
    return map.containsKey(key) ? map.get(key) : defaultValue;
}

def call(String artifactType = 'python_lib_compilation', Map optionalConf = [:]) {
    def valid_artifactTypes = ['python_lib_compilation'];
    String mainBranchName = getValue(optionalConf, 'mainBranchName', 'main');
    String slave_name = getValue(optionalConf, 'slave_name', 'Jenkins_Slave');
    
    float warningsThresholdPylint = getValue(optionalConf, 'warningsThresholdPylint', 8.0);
    int warningsThresholdCoverage = getValue(optionalConf, 'warningsThresholdCoverage', 70);
    
    String writerServerCredId = getValue(optionalConf, 'writerServerCredId', 'my_writer_artifactory_api_key');
    String readerServerCredId = getValue(optionalConf, 'readerServerCredId', 'my_reader_artifactory_api_key');

    if (valid_artifactTypes.contains(artifactType)) {
        pipeline {
            agent {
                node {
                    label slave_name
                }
            }
            stages {
                stage('PreSteps') {
                    parallel {
                        stage('MainBranchPreSteps') {
                            when {
                                anyOf {
                                    branch mainBranchName;
                                    changeRequest target: mainBranchName
                                }
                            }
                            steps {
                                echo 'This is ' + mainBranchName + ' Branch'
                                echo '====installing requirements.txt===='
                                withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId:readerServerCredId,
                                                usernameVariable: 'ARTIFACTORY_USERNAME', passwordVariable: 'ARTIFACTORY_PASSWORD']
                                ]) {
                                    sh """
                                    python3.8 -m pip install virtualenv
                                    python3.8 -m venv python_env
                                    source python_env/bin/activate
                                    python3.8 -m pip install --upgrade pip
                                    python3.8 -m pip install coverage wheel pylint
                                    python3.8 -m pip install -r requirements.txt -i https://${ARTIFACTORY_USERNAME}:${ARTIFACTORY_PASSWORD}@artifactory.my-corp.com/artifactory/api/pypi/pypi-my-corp-release/simple
                                    deactivate
                                    """
                                }
                                echo '====installation done===='
                            }
                        }
                        stage('NonMainBranchPreSteps') {
                            when {
                                not {
                                    anyOf {
                                        branch mainBranchName;
                                        changeRequest target: mainBranchName
                                    }
                                }
                            }
                            steps {
                                echo 'This is Non ' + mainBranchName + ' Branch'
                            }
                        }
                    }
                }
    //            only when to merge to master branch, so not on PR or when new branch is cut
                stage('compilation') {
                    when {
                        not {
                            anyOf {
                                changeRequest target: mainBranchName
                            }
                        }
                        anyOf {
                            branch mainBranchName;
                        }
                    }
                    steps {
                        echo 'compilation..'
                        echo 'merging to main branch'
                        sh """
                            python3.8 -m pip install virtualenv
                            python3.8 -m venv python_env
                            source python_env/bin/activate
                            python3.8 -m pip install coverage wheel pylint
                            python3.8 -m pip install --upgrade pip
                            python3.8 setup.py sdist bdist_wheel
                            deactivate
                        """
                    }
                }
                stage('Test-File-Existence') {
                    parallel {
            //            only when there is a PR
            //            testing if version.txt file exists or not
                        stage('Test-File-Exists') {
                            when {
                                allOf {
                                    changeRequest target: mainBranchName
                                    expression { return fileExists('version.txt') && readFile('version.txt').contains('__version__') }
                                }
                            }
                            steps {
                                echo "checking if version.txt file exists or not"
                                echo "=====file exists===="
                            }
                        }
                        stage('Test-File-Dont-Exists') {
                            when {
                                allOf {
                                    changeRequest target: mainBranchName
                                }
                                not {
                                    expression { return fileExists('version.txt') && readFile('version.txt').contains('__version__') }
                                }
                            }
                            steps {
                                echo "=====Either file does not exists or version is not defined===="
                                sh 'exit 1'
                            }
                        }
                    }
                }

    //            only when there is a PR
    //            testing structure i.e. pylint, coverage, unittest
                stage('Test-Structure') {
                    when {
                        anyOf {
                            changeRequest target: mainBranchName
                        }
                    }

                    parallel {
                        stage('Pylint-test') {
                            when {
                                anyOf {
                                    changeRequest target: mainBranchName
                                }
                            }
                            steps {
                                echo 'Testing..'
                                sh """
                                    python3.8 -m pip install virtualenv
                                    python3.8 -m venv python_env
                                    source python_env/bin/activate
                                    python3.8 -m pip install --upgrade pip
                                    python3.8 -m pip install coverage wheel pylint
                                    python3.8 setup.py sdist bdist_wheel
                                    echo "Pylint warning number: ${warningsThresholdPylint}"
                                    pylint --fail-under=${warningsThresholdPylint} **/*.py>pylint.log
                                    cat pylint.log
                                    deactivate
                                """
                            }
                        }
                        stage('Unit-test') {
                            when {
                                anyOf {
                                    changeRequest target: mainBranchName
                                }
                            }
                            steps {
                                echo 'Testing..'
                                sh """
                                    python3.8 -m pip install virtualenv
                                    python3.8 -m venv python_env
                                    source python_env/bin/activate
                                    python3.8 -m pip install --upgrade pip
                                    python3.8 -m pip install coverage wheel pylint
                                    python3.8 -m unittest tests/*.py
                                    deactivate
                                """
                            }
                        }
                        stage('Coverage-test') {
                            when {
                                anyOf {
                                    changeRequest target: mainBranchName
                                }
                            }
                            steps {
                                echo 'Testing..'
                                sh """
                                    python3.8 -m pip install virtualenv
                                    python3.8 -m venv python_env
                                    source python_env/bin/activate
                                    python3.8 -m pip install --upgrade pip
                                    python3.8 -m pip install coverage wheel pylint
                                    coverage run -m unittest tests/*.py
                                    coverage report -m --fail-under=${warningsThresholdCoverage} **/*.py
                                    deactivate
                                """
                                
                            }
                        }
                    }
                }

    //            deploy only when merged
                stage('Deploy') {
                    when {
                        allOf {
                            branch mainBranchName;
                        }
                    }
                    steps {
                        echo 'Deploying....'
                        withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId:writerServerCredId,
                                        usernameVariable: 'TWINE_USERNAME', passwordVariable: 'TWINE_PASSWORD']
                        ]) {
                            sh """
                                python3.8 -m pip install virtualenv
                                python3.8 -m venv python_env
                                source python_env/bin/activate
                                python3.8 -m pip install --upgrade pip
                                python3.8 -m pip install coverage wheel pylint twine
                                python3.8 -m twine upload --repository-url https://artifactory.corp.adobe.com/artifactory/api/pypi/pypi-cloudtech-release-local dist/* -u ${TWINE_USERNAME} -p ${TWINE_PASSWORD}
                                deactivate
                            """
                        }
                    }
                }
            }
            post {
                always {
                    emailext(subject: '$DEFAULT_SUBJECT', body: '$DEFAULT_CONTENT', attachLog: true, recipientProviders: [[$class: 'CulpritsRecipientProvider'], [$class: 'DevelopersRecipientProvider'], [$class: 'FailingTestSuspectsRecipientProvider'], [$class: 'FirstFailingBuildSuspectsRecipientProvider'], [$class: 'RequesterRecipientProvider'], [$class: 'BuildUserRecipientProvider']], to: '$DEFAULT_RECIPIENTS')
                }
            }
        }
    } else {
        pipeline {
            agent {
                node {
                    label slave_name
                }
            }
            stages {
                stage('InvalidStage') {
                    steps {
                        echo "The build argument provided must one of : ${valid_artifactTypes.join(',')}"
                    }
                }
            }
        }
    }
}