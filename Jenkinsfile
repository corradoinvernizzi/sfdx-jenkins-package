#!groovy

import groovy.json.JsonSlurperClassic

node {

    def SF_CONSUMER_KEY=env.CONNECTED_APP_CONSUMER_KEY_DH
    def SF_USERNAME=env.HUB_ORG_DH
    def SERVER_KEY_CREDENTALS_ID=env.JWT_CRED_ID_DH
    def TEST_LEVEL='RunLocalTests'
    def PACKAGE_NAME='0Ho1p000000GmcKCAS'
    def PACKAGE_VERSION


    


    // -------------------------------------------------------------------------
    // Check out code from source control.
    // -------------------------------------------------------------------------

    stage('checkout source') {
        checkout scm
    }


    // -------------------------------------------------------------------------
    // Run all the enclosed stages with access to the Salesforce
    // JWT key credentials.
    // -------------------------------------------------------------------------

    withCredentials([file(credentialsId: SERVER_KEY_CREDENTALS_ID, variable: 'server_key_file')]) {

        // -------------------------------------------------------------------------
        // Authorize the Dev Hub org with JWT key and give it an alias.
        // -------------------------------------------------------------------------

        stage('Authorize DevHub') {
            rc = command "sfdx force:auth:jwt:grant --clientid ${SF_CONSUMER_KEY} --username ${SF_USERNAME} --jwtkeyfile ${server_key_file} --setdefaultdevhubusername --setalias DevHub"
            if (rc != 0) {
                error 'Salesforce dev hub org authorization failed.'
            }
        }


        // -------------------------------------------------------------------------
        // Create new scratch org to test your code.
        // -------------------------------------------------------------------------

        stage('Create Test Scratch Org') {
            rc = command "sfdx force:org:create --targetdevhubusername DevHub --setdefaultusername --definitionfile config/project-scratch-def.json --setalias ciorg --wait 10 --durationdays 1"
            if (rc != 0) {
                error 'Salesforce test scratch org creation failed.'
            }
        }


        // -------------------------------------------------------------------------
        // Display test scratch org info.
        // -------------------------------------------------------------------------

        stage('Display Test Scratch Org') {
            rc = command "sfdx force:org:display --targetusername ciorg"
            if (rc != 0) {
                error 'Salesforce test scratch org display failed.'
            }
        }


        // -------------------------------------------------------------------------
        // Push source to test scratch org.
        // -------------------------------------------------------------------------

        stage('Push To Test Scratch Org') {
            rc = command "sfdx force:source:push --targetusername ciorg"
            if (rc != 0) {
                error 'Salesforce push to test scratch org failed.'
            }
        }


        // -------------------------------------------------------------------------
        // Run unit tests in test scratch org.
        // -------------------------------------------------------------------------

        stage('Run Tests In Test Scratch Org') {
            rc = command "sfdx force:apex:test:run --targetusername ciorg --wait 10 --resultformat junit --codecoverage --testlevel ${TEST_LEVEL}"
            if (rc != 0) {
                error 'Salesforce unit test run in test scratch org failed.'
            }
        }


        // -------------------------------------------------------------------------
        // Delete test scratch org.
        // -------------------------------------------------------------------------

        stage('Delete Test Scratch Org') {
            rc = command "sfdx force:org:delete --targetusername ciorg --noprompt"
            if (rc != 0) {
                error 'Salesforce test scratch org deletion failed.'
            }
        }


        // -------------------------------------------------------------------------
        // Create package version.
        // -------------------------------------------------------------------------

        stage('Create Package Version') {
            if (isUnix()) {
                output = sh returnStdout: true, script: "sfdx force:package:version:create --package ${PACKAGE_NAME} --installationkeybypass --wait 10 --json --targetdevhubusername DevHub"
            } else {
                output = bat(returnStdout: true, script: "sfdx force:package:version:create --package ${PACKAGE_NAME} --installationkeybypass --wait 10 --json --targetdevhubusername DevHub").trim()
                output = output.readLines().drop(1).join(" ")
            }

            // Wait 5 minutes for package replication.
            sleep 300

            def jsonSlurper = new JsonSlurperClassic()
            def response = jsonSlurper.parseText(output)

            PACKAGE_VERSION = response.result.SubscriberPackageVersionId
            
            response = null

            //echo ${PACKAGE_VERSION}
        }


        // -------------------------------------------------------------------------
        // Create new scratch org to install package to.
        // -------------------------------------------------------------------------

        stage('Create Package Install Scratch Org') {
            rc = command "sfdx force:org:create --targetdevhubusername DevHub --setdefaultusername --definitionfile config/project-scratch-def.json --setalias installorg --wait 10 --durationdays 1"
            if (rc != 0) {
                error 'Salesforce package install scratch org creation failed.'
            }
        }


        // -------------------------------------------------------------------------
        // Display install scratch org info.
        // -------------------------------------------------------------------------

        stage('Display Install Scratch Org') {
            rc = command "sfdx force:org:display --targetusername installorg"
            if (rc != 0) {
                error 'Salesforce install scratch org display failed.'
            }
        }


        // -------------------------------------------------------------------------
        // Install package in scratch org.
        // -------------------------------------------------------------------------

        stage('Install Package In Scratch Org') {
            rc = command "sfdx force:package:install --package ${PACKAGE_VERSION} --targetusername installorg --wait 10"
            if (rc != 0) {
                error 'Salesforce package install failed.'
            }
        }


        // -------------------------------------------------------------------------
        // Run unit tests in package install scratch org.
        // -------------------------------------------------------------------------

        stage('Run Tests In Package Install Scratch Org') {
            rc = command "sfdx force:apex:test:run --targetusername installorg --resultformat junit --codecoverage --testlevel ${TEST_LEVEL} --wait 10"
            if (rc != 0) {
                error 'Salesforce unit test run in pacakge install scratch org failed.'
            }
        }


        // -------------------------------------------------------------------------
        // Delete package install scratch org.
        // -------------------------------------------------------------------------

        stage('Delete Package Install Scratch Org') {
            rc = command "sfdx force:org:delete --targetusername installorg --noprompt"
            if (rc != 0) {
                error 'Salesforce package install scratch org deletion failed.'
            }
        }

        // -------------------------------------------------------------------------
        // Install package in DevHub org.
        // -------------------------------------------------------------------------

        stage('Install Package In DevHub Org') {
            rc = command "sfdx force:package:install --package ${PACKAGE_VERSION} --targetusername DevHub --wait 10"
            if (rc != 0) {
                error 'Salesforce package install failed.'
            }
        }
    }
}

def command(script) {
    if (isUnix()) {
        return sh(returnStatus: true, script: script);
    } else {
        return bat(returnStatus: true, script: script);
    }
}
