pipeline {
    agent any
    
    parameters {
        string(name: 'ORGANIZATION_ID', defaultValue: "${env.ORGANIZATION_ID_01 ?: ''}", description: 'Organization ID')
        string(name: 'ENVIRONMENT_ID', defaultValue: "${env.ENVIRONMENT_ID ?: ''}", description: 'Environment ID')
        string(name: 'ENVIRONMENT_TYPE', defaultValue: 'dev', description: 'Environment type (e.g., dev, qa, prod)')
        string(name: 'REPO_URL', defaultValue: 'git@github.com:aalleva/automated-policies-configuration.git', description: 'Git Repository URL for pipeline code')
        string(name: 'CHECKOUT_BRANCH', defaultValue: 'main', description: 'Branch to checkout from the repository')
    }
    
    environment {
        MULESOFT_API_URL = 'anypoint.mulesoft.com'
        CURL_TIMEOUT = '30'
        CONFIG_FOLDER = 'config'
    }
    
    stages {
        stage('Input Parameters Validation') {
            steps {
                script {
                    if (!params.ORGANIZATION_ID?.trim()) {
                        error "ORGANIZATION_ID is missing or invalid."
                    }
                    if (!params.ENVIRONMENT_ID?.trim()) {
                        error "ENVIRONMENT_ID is missing or invalid."
                    }
                    if (!params.ENVIRONMENT_TYPE?.trim()) {
                        error "ENVIRONMENT_TYPE is missing or invalid."
                    }
                    if (!params.REPO_URL?.trim()) {
                        error "REPO_URL is missing or invalid."
                    }
                    if (!params.CHECKOUT_BRANCH?.trim()) {
                        error "CHECKOUT_BRANCH is missing or invalid."
                    }
                    
                    echo "Validation passed. ORGANIZATION_ID: ${params.ORGANIZATION_ID}, ENVIRONMENT_ID: ${params.ENVIRONMENT_ID}, REPO_URL: ${params.REPO_URL}, CHECKOUT_BRANCH: ${params.CHECKOUT_BRANCH}"
                }
            }
        }
        stage('Checkout') {
            steps {
                script {
                    
                    echo "Cloning repository from URL: ${params.REPO_URL}, branch: ${params.CHECKOUT_BRANCH}"
                    git branch: "${params.CHECKOUT_BRANCH}", url: "${params.REPO_URL}", credentialsId: 'github-ssh-key'
                }
            }
        }
        stage('Fetch Configuration File') {
            steps {
                script {
                    def configFilePath = "${WORKSPACE}/${env.CONFIG_FOLDER}/automated-policies-${params.ENVIRONMENT_TYPE}-conf.json"
                    echo "Reading configuration file: ${configFilePath}"

                    if (!fileExists(configFilePath)) {
                        error "Configuration file not found: ${configFilePath}"
                    }

                    // Read and parse the configuration JSON file
                    def configJsonContent = readFile(configFilePath)
                    env.CONFIG_JSON = configJsonContent
                }
            }
        }
        stage('Fetch Bearer Token') {
            steps {
                withCredentials([string(credentialsId: 'CONNECTED_APP_CLIENT_ID', variable: 'CLIENT_ID'), 
                                 string(credentialsId: 'CONNECTED_APP_CLIENT_SECRET', variable: 'CLIENT_SECRET')]) {
                    script {
                        def tokenResponse = sh(script: """
                            curl --location 'https://${env.MULESOFT_API_URL}/accounts/api/v2/oauth2/token' \
                            --header 'Content-Type: application/json' \
                            --silent \
                            --data '{
                                "client_id": "${CLIENT_ID}",
                                "client_secret": "${CLIENT_SECRET}",
                                "grant_type": "client_credentials"
                            }' \
                            --write-out 'HTTPSTATUS:%{http_code}' --silent --max-time ${env.CURL_TIMEOUT}
                        """, returnStdout: true).trim()
                        def jsonResponse = tokenResponse.replaceAll(/HTTPSTATUS:\d{3}$/, '').trim()
                        def httpStatus = tokenResponse[-3..-1]
                        if (httpStatus == '200') {
                            def json = readJSON text: jsonResponse
                            def accessToken = json.access_token
                            def tokenType = json.token_type
                            env.AUTHORIZATION_TOKEN = "${tokenType} ${accessToken}"
                        } else {
                            error "Error while getting Bearer Token. Status Code: ${httpStatus}. Response: ${jsonResponse}"
                        }
                    }
                }
            }
        }
        stage('List Automated Policies') {
            steps {
                script {
                    def policiesResponseFile = 'policies_response.json'
                    def policiesResponse = sh(script: """
                      curl --location 'https://${env.MULESOFT_API_URL}/apimanager/api/v1/organizations/${params.ORGANIZATION_ID}/automated-policies?environmentId=${params.ENVIRONMENT_ID}' \
                      --header 'Authorization: ${env.AUTHORIZATION_TOKEN}' \
                      --header 'x-anypnt-org-id: ${params.ORGANIZATION_ID}' \
                      --header 'x-anypnt-env-id: ${params.ENVIRONMENT_ID}' \
                      --silent --max-time ${env.CURL_TIMEOUT} --write-out 'HTTPSTATUS:%{http_code}' --output ${policiesResponseFile}
                  """, returnStdout: true).trim()
                    
                    def httpStatus = policiesResponse[-3..-1]
                    def jsonResponse = readFile(policiesResponseFile)
                    
                    if (httpStatus == '200') {
                        echo "Automated Policies fetched successfully."
                        env.POLICIES_JSON_RESPONSE = jsonResponse
                    } else {
                        error "Error fetching policies. Status code: ${httpStatus}. Response: ${jsonResponse}"
                    }
                }
            }
        }
        stage('Check Existing Policies') {
            steps {
                script {
                    def configJson = readJSON text: env.CONFIG_JSON
                    def policiesJson = readJSON text: env.POLICIES_JSON_RESPONSE
                    def existingPolicies = policiesJson.automatedPolicies
        
                    def order = 1
        
                    configJson.each { policy ->
                        def matchingPolicy = existingPolicies.find { existing ->
                            existing.groupId == policy.groupId &&
                            existing.assetId == policy.assetId &&
                            existing.assetVersion == policy.assetVersion
                        }
                        if (matchingPolicy) {
                            echo "Policy exists: ${policy.assetId}. ID: ${matchingPolicy.id}. Order: ${order}"
                            policy.policyId = matchingPolicy.id 
                            
                            def configData = groovy.json.JsonOutput.toJson(policy.configurationData).replaceAll("'", "'\\\\''")
                            def updateResponse = sh(script: """
                                curl --location --request PATCH 'https://${env.MULESOFT_API_URL}/apimanager/api/v1/organizations/${params.ORGANIZATION_ID}/automated-policies/${matchingPolicy.id}' \
                                --header 'Authorization: ${env.AUTHORIZATION_TOKEN}' \
                                --header 'x-anypnt-org-id: ${params.ORGANIZATION_ID}' \
                                --header 'x-anypnt-env-id: ${params.ENVIRONMENT_ID}' \
                                --header 'Content-Type: application/json' \
                                --silent \
                                --data '{
                                    "ruleOfApplication": {
                                        "technologies": [
                                            "flexGateway"
                                        ],
                                        "environmentId": "${params.ENVIRONMENT_ID}",
                                        "organizationId": "${params.ORGANIZATION_ID}"
                                    },
                                    "groupId": "${policy.groupId}",
                                    "assetId": "${policy.assetId}",
                                    "assetVersion": "${policy.assetVersion}",
                                    "order": ${order},
                                    "configurationData": ${configData}
                                }' \
                                --write-out '%{http_code}' --max-time ${env.CURL_TIMEOUT} --output /dev/null
                            """, returnStdout: true).trim()
        
                            if (updateResponse == '200') {
                                echo "Policy updated successfully: ${policy.assetId} with Order: ${order}"
                            } else {
                                error "Error updating policy: ${policy.assetId}. Response code: ${updateResponse}"
                            }
                            
                        } else {
                            echo "Policy does not exist: ${policy.assetId}. Order: ${order}"
                            
                            // Crear la política si no existe
                            def configData = groovy.json.JsonOutput.toJson(policy.configurationData).replaceAll("'", "'\\\\''")
                            def createResponse = sh(script: """
                                curl --location 'https://${env.MULESOFT_API_URL}/apimanager/api/v1/organizations/${params.ORGANIZATION_ID}/automated-policies' \
                                --header 'Authorization: ${env.AUTHORIZATION_TOKEN}' \
                                --header 'x-anypnt-org-id: ${params.ORGANIZATION_ID}' \
                                --header 'x-anypnt-env-id: ${params.ENVIRONMENT_ID}' \
                                --header 'Content-Type: application/json' \
                                --silent \
                                --data '{
                                    "ruleOfApplication": {
                                        "technologies": [
                                            "flexGateway"
                                        ],
                                        "environmentId": "${params.ENVIRONMENT_ID}",
                                        "organizationId": "${params.ORGANIZATION_ID}"
                                    },
                                    "groupId": "${policy.groupId}",
                                    "assetId": "${policy.assetId}",
                                    "assetVersion": "${policy.assetVersion}",
                                    "order": ${order},
                                    "configurationData": ${configData}
                                }' \
                                --write-out '%{http_code}' --max-time ${env.CURL_TIMEOUT} --output /dev/null
                            """, returnStdout: true).trim()
        
                            if (createResponse == '201') {
                                echo "Policy created successfully: ${policy.assetId} with Order: ${order}"
                            } else {
                                error "Error creating policy: ${policy.assetId}. Response code: ${createResponse}"
                            }
                            
                        }
                        
                        order++
                    }
                }
            }
        }
    }

    post {
      always {
          sh "rm -f policies_response.json"
      }
    }
}
