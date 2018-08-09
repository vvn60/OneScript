
pipeline {
    
    agent none

    environment {
        ReleaseNumber = 21
        outputEnc = '65001'
    }

    stages {
        stage('Windows Build') {
            agent { label 'windows' }

            // пути к инструментам доступны только когда
            // нода уже определена
            environment {
                NugetPath = "${tool 'nuget'}"
                OneScriptDocumenter = "${tool 'documenter'}"
                StandardLibraryPacks = "${tool 'os_stdlib'}"
            }

            steps {
                
                // в среде Multibranch Pipeline Jenkins первращает имена веток в папки
                // а для веток Gitflow вида release/* экранирует в слэш в %2F
                // При этом MSBuild, видя urlEncoding, разэкранирует его обратно, ломая путь (появляется слэш, где не надо)
                //
                // Поэтому, применяем костыль с кастомным workspace
                // см. https://issues.jenkins-ci.org/browse/JENKINS-34564
                //
                // А еще Jenkins под Windows постоянно добавляет в конец папки какую-то мусорную строку.
                // Для этого отсекаем все, что находится после последнего дефиса
                // см. https://issues.jenkins-ci.org/browse/JENKINS-40072
                
                ws(env.WORKSPACE.replaceAll("%", "_").replaceAll(/(-[^-]+$)/, ""))
                {
                    step([$class: 'WsCleanup'])
					checkout scm

                    bat 'set'
                    withSonarQubeEnv('silverbulleters') {
                        script {
                            def sqScannerMsBuildHome = tool 'sonar-scanner for msbuild';
                            sqScannerMsBuildHome = sqScannerMsBuildHome + "\\SonarScanner.MSBuild.exe";
                            def sonarcommandStart = "@" + sqScannerMsBuildHome + " begin /k:1script /n:OneScript /v:\"1.0.${env.ReleaseNumber}\"";
                            def makeAnalyzis = true
                            if (env.BRANCH_NAME == "develop") {
                                echo 'Analysing develop branch'
                                withCredentials([string(credentialsId: 'opensonar_token', variable: 'sonarToken')]) {
                                    sonarcommandStart = sonarcommandStart + " /d:sonar.login=${sonarToken}"
                                }
                            } else if (env.BRANCH_NAME.startsWith("PR-")) {
                                // Report PR issues           
                                def PRNumber = env.BRANCH_NAME.tokenize("PR-")[0]
                                def gitURLcommand = 'git config --local remote.origin.url'
                                def gitURL = ""
                                
                                if (isUnix()) {
                                    gitURL = sh(returnStdout: true, script: gitURLcommand).trim() 
                                } else {
                                    gitURL = bat(returnStdout: true, script: gitURLcommand).trim() 
                                }
                                
                                def repository = gitURL.tokenize("/")[2] + "/" + gitURL.tokenize("/")[3]
                                repository = repository.tokenize(".")[0]
                                withCredentials([string(credentialsId: 'GithubOAUTHToken_ForSonar', variable: 'githubOAuth')]) {
                                    sonarcommandStart = sonarcommandStart + " /d:sonar.analysis.mode=issues /d:sonar.github.pullRequest=${PRNumber} /d:sonar.github.repository=${repository} /d:sonar.github.oauth=${githubOAuth}"
                                }
                            } else {
                                makeAnalyzis = false
                            }

                            if (makeAnalyzis) {
                                bat "${sonarcommandStart}"
                            }
                            bat "chcp $outputEnc > nul\r\n\"${tool 'MSBuild'}\" src/1Script.sln /t:restore"
							bat "chcp $outputEnc > nul\r\n\"${tool 'MSBuild'}\" Build.csproj /t:CleanAll;PrepareDistributionContent"
                            if (makeAnalyzis) {
                                bat "${sqScannerMsBuildHome} end"
                            }
                        }
                    }

                    stash includes: 'tests, built/**', name: 'buildResults'
                }
           }

        }

        stage('VSCode debugger Build') {
            agent {
                docker {
                    image 'node'
                    label 'linux'
                }
            }

            steps {
                unstash 'buildResults'
                sh 'npm install vsce'
                script {
                    def vsceBin = pwd() + "/node_modules/.bin/vsce"
                    sh "cd built/vscode && ${vsceBin} package"
                    archiveArtifacts artifacts: 'built/vscode/*.vsix', fingerprint: true
                    stash includes: 'built/vscode/*.vsix', name: 'vsix' 
                }
            }
        }

        stage('Windows testing') {
            agent { label 'windows' }

            steps {
                ws(env.WORKSPACE.replaceAll("%", "_").replaceAll(/(-[^-]+$)/, ""))
                {
                    dir('install/build'){
                        deleteDir()
                    }
                    unstash 'buildResults'
                    bat "chcp $outputEnc > nul\r\n\"${tool 'MSBuild'}\" Build.csproj /t:xUnitTest"

                    junit 'tests/tests.xml'
                }
            }
        }

        stage('Linux testing') {

            agent { label 'master' }

            steps {
                
                dir('install/build'){
                    deleteDir()
                }
                
                unstash 'buildResults'

                sh '''\
                if [ ! -d lintests ]; then
                    mkdir lintests
                fi

                rm lintests/*.xml -f
                cd tests
                mono ../built/tmp/bin/oscript.exe testrunner.os -runall . xddReportPath ../lintests || true
                exit 0
                '''.stripIndent()

                junit 'lintests/*.xml'
                archiveArtifacts artifacts: 'lintests/*.xml', fingerprint: true
            }



        }
        
        stage('Packaging') {
            agent { label 'windows' }

            environment {
                InnoSetupPath = "${tool 'InnoSetup'}"
            }
            
            steps {
                ws(env.WORKSPACE.replaceAll("%", "_").replaceAll(/(-[^-]+$)/, ""))
                {
                    dir('built'){
                        deleteDir()
                    }
                    
                    unstash 'buildResults'
                    script
                    {
                        if (env.BRANCH_NAME == "preview") {
                            echo 'Building preview'
                            bat "chcp $outputEnc > nul\r\n\"${tool 'MSBuild'}\" Build.csproj /t:CreateDistributions /p:Suffix=-pre%BUILD_NUMBER%"
                        }
                        else{
                            bat "chcp $outputEnc > nul\r\n\"${tool 'MSBuild'}\" Build.csproj /t:CreateDistributions"
                        }
                    }
                    archiveArtifacts artifacts: 'built/**', fingerprint: true
                    stash includes: 'built/**', name: 'winDist'
                }
            }
        }

        stage ('Packaging DEB and RPM') {
            agent { label 'master' }

            steps {

                dir('built'){
                    deleteDir()
                }
                checkout scm
                unstash 'buildResults'

                sh '''
                cd install
                chmod +x prepare-build.sh
                chmod +x deb-build.sh
                chmod +x rpm-build.sh

                sh ./prepare-build.sh
                
                DISTPATH=`pwd`/built/tmp
                
                sh ./deb-build.sh $DISTPATH
                sh ./rpm-build.sh $DISTPATH
                '''.stripIndent()
                
                archiveArtifacts artifacts: 'output/*', fingerprint: true
                stash includes: 'output/*', name: 'linDist'
                
            }

        }

        stage ('Publishing night-build') {
            when { anyOf {
				branch 'develop';
				branch 'release/*'
				}
			}
			
            agent { label 'master' }

            steps {
                
                unstash 'winDist'
                unstash 'linDist'
                unstash 'vsix'
                
                sh '''
                if [ -d "targetContent" ]; then
                    rm -rf targetContent
                fi
                mkdir targetContent
                mv -t targetContent built/*.exe built/*.zip built/vscode/*.vsix
                mv output/*.rpm targetContent/
                mv output/*.deb targetContent/

                TARGET="/var/www/oscript.io/download/versions/night-build/"

                cd targetContent
                sudo rsync -rv --delete --exclude mddoc*.zip --exclude *.src.rpm . $TARGET
                rm -rf targetContent
                '''.stripIndent()
            }
        }
                
        stage ('Publishing master') {
            when { branch 'master' }
                
            agent { label 'master' }

            steps {
                
                unstash 'winDist'
                unstash 'linDist'
                unstash 'vsix'
                
                sh """
                if [ -d "targetContent" ]; then
                    rm -rf targetContent
                fi
                mkdir targetContent
                mv -t targetContent built/*.exe built/*.zip built/vscode/*.vsix
                mv output/*.rpm targetContent/
                mv output/*.deb targetContent/

                cd targetContent
                TARGET="/var/www/oscript.io/download/versions/latest/"
                sudo rsync -rv --delete --exclude mddoc*.zip --exclude *.src.rpm . \$TARGET
                
                TARGET="/var/www/oscript.io/download/versions/1_0_$ReleaseNumber/"
                sudo rsync -rv --delete --exclude mddoc*.zip --exclude *.src.rpm . \$TARGET

                """.stripIndent()
            }
        }

    }
    
}