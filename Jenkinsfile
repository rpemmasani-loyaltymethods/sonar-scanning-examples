pipeline {
    agent any
    
    parameters {
        // Parameter to specify the branch name, default is 'main'
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'Branch to build')
        string(name: 'SONAR_PROJECT_KEY', description: 'Unique identifier')
        string(name: 'SONAR_PROJECT_NAME', description: 'Name of the project')
        choice(name: 'QUALITY_GATE', choices: ['Sonar way', 'Default-Quality-Gate', 'Main-Quality-Gate', 'Feature-Quality-Gate'], description: 'Which quality gate you wanted apply..')
        choice(name: 'GATE_CHOICE', choices: ['select', 'set_as_default'], description: 'Applying qualitygate as select/default for project based on choice')
    }

    environment {
        SONARQUBE_SERVER = 'Sonarqube-8.9.2'
        SONARQUBE_URL = "https://sonarqube.devops.lmvi.net/"
        SONARQUBE_TOKEN = credentials('SONARQUBE_TOKEN')
    }

    stages {
        stage('Git Checkout') {
            steps {
                // Dynamically use the branch name from the parameter
                git branch: "${params.BRANCH_NAME}", changelog: false, poll: false, url: 'https://github.com/rpemmasani-loyaltymethods/sonar-scanning-examples.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def branchName = "${params.BRANCH_NAME}"
                    def qualityGate = "${params.QUALITY_GATE}"

                    // Get SonarScanner home directory using Jenkins tool (Correct tool name 'Sonarscanner')
                    def scannerHome = tool name: 'Sonarscanner'

                    // Run SonarScanner for analysis (no pom.xml)
                    withSonarQubeEnv('SonarQube') {  // Set SonarQube environment
                        withCredentials([string(credentialsId: 'SONARQUBE_TOKEN', variable: 'SONARQUBE_TOKEN')]) {
                            sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=${params.SONAR_PROJECT_KEY} \
                            -Dsonar.projectName=${params.SONAR_PROJECT_NAME} \
                            -Dsonar.host.url=${SONARQUBE_URL} \
                            -Dsonar.login=${SONARQUBE_TOKEN} \
                            -Dsonar.exclusions=vendor/**,resources/**,**/*.java \
                            -Dsonar.ws.timeout=600
                            """
                        }
                    }


                    // Apply quality gate setting via API
                    def gatename = params.GATE_CHOICE == 'set_as_default' ? 'name' : 'gateName'

                    withCredentials([string(credentialsId: 'SonarToken', variable: 'SonarToken')]) {
                        // Set quality gate via SonarQube API (with proper credentials and URL)
                        sh """
                        curl --header 'Authorization: Basic ${SonarToken}' \
                        --location '${SONARQUBE_URL}api/qualitygates/${params.GATE_CHOICE}?projectKey=${params.SONAR_PROJECT_KEY}' \
                        --data-urlencode '${gatename}=${qualityGate}'
                        """
                    }

                    // Sleep for 2 minutes after SonarQube analysis
                    echo 'Sleeping for 2 minutes after SonarQube analysis...'
                    sleep(time: 2, unit: 'MINUTES')
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    // Define SonarQube project status API URL
                    def sonarUrl = "${SONARQUBE_URL}api/qualitygates/project_status?projectKey=${params.SONAR_PROJECT_KEY}"

                    withCredentials([string(credentialsId: 'SONARQUBE_TOKEN', variable: 'SONARQUBE_TOKEN')]) {
                        // Fetch the quality gate status and write it to a file
                        sh """
                            curl -s -u ${SONARQUBE_TOKEN}: ${sonarUrl} > sonar_status.json
                        """
                        // Groovy script to check the quality gate status from the JSON file
                        def sonarStatusJson = readFile('sonar_status.json')
                        def sonarData = new groovy.json.JsonSlurper().parseText(sonarStatusJson)
                        echo "SonarQube Response Json: ${sonarData}"
                        // Extract relevant information from the JSON
                        def sonarStatus = sonarData?.projectStatus?.status ?: 'Unknown'
                        echo "SonarQube Quality Gate Status: ${sonarStatus}"

                        if (sonarStatus != 'OK') {
                            echo "Quality Gate failed! SonarQube status: ${sonarStatus}"
                            currentBuild.result = 'FAILURE'  // Mark the build as failed
                            error "Quality Gate Failed!"  // Terminate the build with failure
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed.'
        }
        always {
            cleanWs()
        }
    }
}

