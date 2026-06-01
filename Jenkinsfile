pipeline {
            agent {
                node {

                        label 'roboshop'

                }
            }

            environment {

                appversion = ""
                acc_id = "442940292368"
            }
                // options { disableConcurrentBuilds() }
                //  parameters {
                //     string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')

                //     text(name: 'BIOGRAPHY', defaultValue: '', description: 'Enter some information about the person')

                //     booleanParam(name: 'DEPLOY', defaultValue: true, description: 'Toggle this value')

                //     choice(name: 'CHOICE', choices: ['One', 'Two', 'Three'], description: 'Pick something')

                //     password(name: 'PASSWORD', defaultValue: 'SECRET', description: 'Enter a password')
                // }
    
            stages {

                stage('Read Version') {

                    steps {

                        script {

                                    // Read the file from the workspace
                            def packageJson = readJSON file: 'package.json'
                            
                            // Access properties
                            appversion = packageJson.version
                            // def name = packageJson.name
                            
                            echo "Building version ${appversion}"

                        }

                    }


                }


                stage('Installdependencies') {
                    steps {
                        script {

                                sh """    
                                        

                                        npm install
                                    

                                """    
                        }
                    }
                }

                stage('Unit tests'){
            steps {
                script {
                    sh """
                        npm test
                    """
                }
            }
        }
                stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool name: 'sonar-8'

                    withSonarQubeEnv('sonar-server') {
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
                stage('dockerbuild') {
                    steps {
                        script {

                                withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                            
                        
                                sh """    
                                aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${acc_id}.dkr.ecr.us-east-1.amazonaws.com
                                    docker build -t ${acc_id}.dkr.ecr.us-east-1.amazonaws.com/roboshop/catalogue:${appversion} .
                                    docker push ${acc_id}.dkr.ecr.us-east-1.amazonaws.com/roboshop/catalogue:${appversion}
                           
                                 """    
                        }
                    }
                }

                }
              
            }

}         
    
    
    