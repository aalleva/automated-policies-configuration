pipeline {
    agent any
    parameters {
        string(name: 'ORGANIZATION_ID', defaultValue: "${env.ORGANIZATION_ID ?: ''}", description: 'Organization ID')
        string(name: 'ENVIRONMENT_ID', defaultValue: "${env.ENVIRONMENT_ID ?: ''}", description: 'Environment ID')
        text(name: 'POLICIES_JSON', defaultValue: '', description: 'Paste the JSON policies configuration file here')
    }
    environment {
        MULESOFT_API_URL = 'eu1.anypoint.mulesoft.com'
        CURL_TIMEOUT = '30'
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
                    if (!params.POLICIES_JSON?.trim()) {
                        error "POLICIES_JSON is missing or invalid."
                    }
                    
                    echo "Validation passed. ORGANIZATION_ID: ${params.ORGANIZATION_ID}, ENVIRONMENT_ID: ${params.ENVIRONMENT_ID}"
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
                    def policiesResponse = sh(script: """
                      curl --location 'https://${env.MULESOFT_API_URL}/apimanager/api/v1/organizations/${params.ORGANIZATION_ID}/automated-policies?environmentId=${params.ENVIRONMENT_ID}' \
                      --header 'Authorization: ${env.AUTHORIZATION_TOKEN}' \
                      --header 'x-anypnt-org-id: ${params.ORGANIZATION_ID}' \
                      --header 'x-anypnt-env-id: ${params.ENVIRONMENT_ID}' \
                      --silent --max-time ${env.CURL_TIMEOUT} --write-out 'HTTPSTATUS:%{http_code}' --output policies_response.json
                  """, returnStdout: true).trim()
                    
                    def httpStatus = policiesResponse[-3..-1]
                    def jsonResponse = readFile('policies_response.json')
                    
                    if (httpStatus == '200') {
                        def policiesJson = readJSON text: jsonResponse
                        echo "Automated Policies found:"
                        for (policy in policiesJson.automatedPolicies) {
                            echo "Policy ID: ${policy.id}, Asset ID: ${policy.assetId}, Version: ${policy.assetVersion}"
                        }
                        env.POLICIES_TO_DELETE = policiesJson.automatedPolicies*.id.join(",")
                    } else {
                        error "Error fetching policies. Status code: ${httpStatus}. Response: ${jsonResponse}"
                    }
                }
            }
        }
        stage('Delete All Automated Policies') {
            when {
                expression {
                    return env.POLICIES_TO_DELETE != ""
                }
            }
            steps {
                script {
                    def policyIds = env.POLICIES_TO_DELETE.split(",")
                    for (policyId in policyIds) {
                        echo "Deleting Policy ID: ${policyId}"
                        def deleteResponse = sh(script: """
                            curl --location --request DELETE 'https://${env.MULESOFT_API_URL}/apimanager/api/v1/organizations/${params.ORGANIZATION_ID}/automated-policies/${policyId}' \
                            --header 'Authorization: ${env.AUTHORIZATION_TOKEN}' \
                            --header 'x-anypnt-org-id: ${params.ORGANIZATION_ID}' \
                            --header 'x-anypnt-env-id: ${params.ENVIRONMENT_ID}' \
                            --write-out '%{http_code}' --silent --max-time ${env.CURL_TIMEOUT} --output /dev/null
                        """, returnStdout: true).trim()
                        if (deleteResponse == '204') {
                            echo "Policy ID ${policyId} deleted successfully."
                        } else {
                            error "Error deleting Policy ID ${policyId}: ${deleteResponse}"
                        }
                    }
                }
            }
        }
        stage('Create New Policies from JSON') {
            steps {
                script {
                    def configJson = readJSON text: params.POLICIES_JSON
                    for (policy in configJson) {
                        echo "Creating Policy: Asset ID = ${policy.assetId}, Version = ${policy.assetVersion}"
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
                                "configurationData": ${configData}
                            }' \
                            --write-out '%{http_code}' --max-time ${env.CURL_TIMEOUT} --output /dev/null
                        """, returnStdout: true).trim()
                        if (createResponse == '201') {
                            echo "Policy created successfully: ${policy.assetId}"
                        } else {
                            error "Error creating policy: ${policy.assetId}. Response code: ${createResponse}"
                        }
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
