#!/usr/bin/groovy
import groovy.json.JsonOutput
import io.vz.pipelinejob.Model.*
import io.vz.pipelinejob.Helpers.*

def call(body) {
    def validateReport
    def deployServiceInfoList
    def pipelineParams = [:]
    def pipelineStages = [:]
    def paramJson = [:]
    def errMsg
    def parameters
    def serviceList
    def passiveDeploy
    def deployedServiceSitB
    def deployedServiceSitG
    def envTag
    def app_configMapList = ''
    def passiveCluster = [:]
    def deployConfigProperty
    def dEnv = ''
    def envIndex = ''
    def stageRun = true
    def updateTag = true
    def ec2PipelineHelper = new EC2PipelineHelper(this)
    def adhocFileName= ''
    String pCategory=''
    def activePassiveSync
    pipeline {
        agent {
            //label /*'nsa-soe-blue-hivv-npgw-2' Unified-Pieline */ 'VZW.SCM-18' //'unifiedpipeline-slave'
            label 'unifiedpipeline-CD-DeployAgent'
        }
        stages {
            stage("Init"){
                steps{
                    script{
                        try{
                            wrap([$class: 'BuildUser']) {
                                env.BUILD_USER = sh ( script: 'echo "${BUILD_USER}"', returnStdout: true).trim()
                                cleanWs()
                            }               
                            echo "Tier Selected --> ${deployment_trigger}"
                            echo "stages Selected --> ${stages}"  
                            echo "adhocParameters --> ${adhocParameters}"
                            echo "deployEnv --> ${deployEnv}"
                            echo "releaseId --> ${releaseId}"
                            echo "deployType --> ${deployType}"
                            echo "gitGroup --> ${gitGroup}"
                            echo "vsad --> ${vsad}"
                            echo "adhoc projectCategory --> ${projectCategory}"
                            echo "adhoc SRI_DEPLOY --> ${SRI_DEPLOY}"
                            echo "full_deployment --> ${full_deployment}"
                            echo "skipActivePassiveSync --> ${skipActivePassiveSync}"
                            if(deployment_trigger == "all_stage"){
                                deployment_trigger = "schedule"
                            }
                            if (deployment_trigger == "adhoc"){
                                deployEnv = deployEnv.replace(",", "")
                                releaseId = releaseId.replace(",", "")
                                deployType = deployType.replace(",", "")
                                if (deployType == "EC2-AEM") {
                                    deployType = "EC2"
                                }
                                pCategory = projectCategory.replace(",", "")
                                env.PROJECT_CATEGORY = pCategory
                                SRI_DEPLOY = SRI_DEPLOY.replace(",", "")
                                env.SRI_DEPLOY = SRI_DEPLOY
                                vsad = vsad.replace(",", "")
                                if ("${adhocParameters}"){                                                           
                                    def paramsList = adhocParameters.split(",")
                                    echo "paramsList => ${paramsList}"
                                    if(paramsList.size()>0){
                                        for (int k=0; k < paramsList.size(); k++){
                                            echo "k -> ${k}"
                                            //echo "paramsList -> ${paramsList[k]}"
                                            paramJson."${paramsList[k]}" = paramsList[k+1].toString()
                                            echo"paramJson --> ${paramJson}"
                                            k = k+1                                        
                                        }
                                    }
                                }
                                pipelineStages."service_Deploy" = stages.contains("service_Deploy")
                                //pipelineStages."update_DeployStatus" = stages.contains("update_DeployStatus")
                                
                                //arf-composite:arf-composite-2eb9cbb9-127521-07212023122450.jar,accessory-composite:accessory-composite-27c2c9e-138727-08012023103355.jar
                                //myb-qa2bg-msmbt.ebiz.verizon.com|arf-composite:arf-composite-2eb9cbb9-127521-07212023122450.jar,accessory-composite:accessory-composite-27c2c9e-138727-08012023103355.jar
                                def tempList
                                if(stages.contains("service_Deploy")){
                                    if(!paramJson.serviceList){
                                        pipelineStages."fetch_deployList" = 'true'                                                                                                                                
                                    }
                                    else{
                                        def temp
                                        def host
                                        try{
                                            tempList = paramJson.serviceList.split("\\|\\|")
                                            if(tempList.size() > 1){
                                                temp = tempList[1].split("\\|")
                                                host = tempList[0]
                                            //println(temp)
                                            }
                                            else{
                                                temp = tempList[0].split("\\|")
                                            }
                                        }
                                        catch(Exception ex)
                                        {
                                            echo "exception: ${ex}"
                                        } 
                                        def reStructureServiceList=''
                                        
                                        def tempService
                                        temp.each{item -> 
                                            tempService = item.split(":")
                                            if(tempService[0].contains("/")){
                                                def mmTempService
                                                mmTempService = tempService[0].split("/")
                                                reStructureServiceList += "${mmTempService[1]}:/${tempService[0]}/${tempService[1]},"
                                                
                                            }
                                            else{
                                                reStructureServiceList += "${tempService[0]}:/${tempService[0]}/${tempService[1]},"
                                            }
                                            
                                        }
                                        if(host){
                                            reStructureServiceList = "${host}|${reStructureServiceList}"
                                        }
                                        else{
                                            reStructureServiceList = "${reStructureServiceList}"
                                        }
                                        reStructureServiceList = reStructureServiceList.substring(0, reStructureServiceList.length() -1)
                                        paramJson.serviceList = reStructureServiceList
                                        echo "${paramJson.serviceList}"
                                        pipelineStages."fetch_deployList" = 'false'
                                        pipelineParams."action" = "deploy"
                                        pipelineParams."dispatcherPath" = ""
                                        passiveDeploy = [:]
                                        passiveDeploy."serviceListToDeploy" = paramJson.serviceList //pipelineParams.serviceList
                                    }

                                    pipelineStages."update_DeployStatus" = 'true'
                                }
                                if(stages.contains("service_dispatcher")){
                                    pipelineParams."dispatcherPath" = paramJson.dispatcherPath
                                    pipelineStages."service_dispatcher" = 'true'
                                    if(stages.contains("service_Deploy")) {
                                    pipelineParams."action" = "deploy,dispatcher"
                                    pipelineStages."service_dispatcher" = 'false'
                                    } else{
                                        pipelineParams."action" = "dispatcher"
                                        pipelineParams."serviceList" = ""
                                        pipelineParams."DeploymentType" = "EC2-AEM"
                                    }
                                }   
                                if(stages.contains("update_DeployStatus")){
                                    adhocFileName = paramJson.build_info
                                    pipelineStages."fetch_deployList" = 'true'
                                }
								if(stages.contains("config_Deploy")) {
									pipelineStages."config_Deploy" = 'true'
								} 
                                else{   
                                pipelineStages."fetch_deployList" = 'true'                                                       
                                dEnv =  deployEnv.toLowerCase()
                                envIndex= dEnv.trim().reverse().take(1)
                                dEnv = dEnv.substring(0, dEnv.length() -1);
                                deployConfigProperty = ec2PipelineInit.getDeploymentConfigs(gitGroup, deployEnv, deployType, '', pCategory)
                                
                                echo "pipelineStages => ${pipelineStages}"
                                echo "pipelineParams ==> ${pipelineParams}"
                                }
                            //}
                            }
                            else if (deployment_trigger == "schedule"){
                                def scheduleParam = scheduleParameters.substring(0, scheduleParameters.length() -1)
                                scheduleParam = scheduleParam.replace("\\", "")
                                echo "scheduleParams: " + scheduleParam
                                paramJson = readJSON text: scheduleParam
                                println("paramJson => ${paramJson}")
                                dEnv =  paramJson.deployEnv.toLowerCase() 
                                if (paramJson.deployType == "EC2-AEM") {
                                    paramJson.deployType = "EC2"
                                }
                                envIndex= dEnv.trim().reverse().take(1)
                                dEnv = dEnv.substring(0, dEnv.length() -1)
                                try
                                {
                                    if(paramJson.projectCategory)
                                    {
                                        pCategory = paramJson.projectCategory
                                    }
                                    else
                                    {
                                        pCategory = 'NA'
                                    }

                                }
                                catch(Exception ex){
                                    steps.log.info("this project doesnot have project Category")
                                    pCategory = 'NA'
                                }
                                env.PROJECT_CATEGORY = pCategory
                                // Below stages will be added through deploy config property 
                                deployConfigProperty = ec2PipelineInit.getDeploymentConfigs(paramJson.gitGroup, paramJson.deployEnv, paramJson.deployType, '', pCategory)
                                println("deployConfigProperty => ${deployConfigProperty}")
                                
                                pipelineStages."service_Deploy" = 'true'                                
                                pipelineStages."fetch_deployList" = 'true'   
								pipelineStages."config_Deploy" = 'true'                             
                                pipelineStages."update_DeployStatus" = 'true'
                                println("trigger schedule >> deployConfigProperty => ${full_deployment}")
                                pipelineParams."full_deployment" = full_deployment
                                println("pipelineStages => ${pipelineStages}")
                            }
                            println("paramJson => ${paramJson}")
                            env.TRIGGER_TYPE = deployment_trigger
                            pipelineParams."user" = deployConfigProperty."user"  
                            pipelineParams."infra_type" = deployConfigProperty.infra_type ?: 'ec2' 
                            pipelineParams."timestamp" = deployConfigProperty.timestamp ?: ""        
                            pipelineParams."config_yaml_file_name" = deployConfigProperty."config_yaml_file_name"
                            pipelineParams."releaseId" = releaseId ?: paramJson.releaseId
                            pipelineParams."deployEnv" = dEnv.toUpperCase() //?: paramJson.deployEnv
                            pipelineParams."envIndex" = envIndex
                            pipelineParams."gitGroup" = gitGroup.toUpperCase() ?: paramJson.gitGroup.toUpperCase()
                            pipelineParams."vsad" = paramJson.vsad ?: vsad
                            pipelineParams."vsad" = pipelineParams.vsad.toUpperCase()
                            pipelineParams."deployType" = deployType ?: paramJson.deployType
                            pipelineParams."SRI_DEPLOY" = env.SRI_DEPLOY ?: ""
							pipelineParams."config_Deploy" config_Deploy ?: paramJson.config_Deploy
                            try{            
                                if(deployConfigProperty.script_dir){
                                    pipelineParams."script_dir" = deployConfigProperty.script_dir
                                }
                                else{
                                    pipelineParams."script_dir" = "/app"
                                }
                                //onevz-cloud-modules,onevz-service-modules,onevz-ui-event-modules
                            }
                            catch(Exception ex){
                                pipelineParams."script_dir" = "/app"
                            }
                            try
                            {
                                env.DEPLOY_MODULES = deployConfigProperty.pos_deploy_modules
                            }
                            catch(Exception ex){
                                env.DEPLOY_MODULES = "all"
                            }

                            println("2. pipelineParams => ${pipelineParams}")
                            println("dEnv => ${dEnv}")                         
                            if(pipelineParams.deployEnv.contains('DIT')){
                                pipelineParams."artifactoryHost" = deployConfigProperty.artifactory_info["Non-prod-host"]
                                pipelineParams."artifactoryRepo" = deployConfigProperty.artifactory_info["Non-prod-repo"]
                            }
                            else if(pipelineParams.deployEnv.contains('SIT')){
                                pipelineParams."artifactoryHost" = deployConfigProperty.artifactory_info["Non-prod-host"]
                                pipelineParams."artifactoryRepo" = deployConfigProperty.artifactory_info["Non-prod-repo"]
                                bgTraffic = bgTraffic.replace(",", "")
                                
                                println("bgTraffic => ${bgTraffic}")
                                pipelineParams."bgTraffic" =  bgTraffic ?: 'none'
                                activePassiveSync = (pipelineParams.bgTraffic == 'none') ? 'true' : 'false'
                                println("activePassiveSync => ${activePassiveSync}")
                            }                         
                            env.FILE_NAME = "${pipelineParams.deployEnv}_${env.BUILD_NUMBER}.json"
                            echo "pipelineParams => ${pipelineParams}"
                            //logic to add config property based on ENV
                            println("pipelineParams => ${pipelineParams}") 
                        }
                        catch(Exception ex){
                            errMsg = "failed due to exception: " + ex.getMessage()
                            error errMsg
                        }
                    }
                }
            }
            stage("Get passive Server"){
                when {
                    expression { pipelineParams.deployEnv.contains('SIT') && stageRun == true }
                }
                steps{
                    script{                        
                        
                        def bgIndex= ''
                        if(deployment_trigger != "adhoc" && activePassiveSync.toString() == 'true'){         
                            passiveCluster = ec2PipelineHelper.getDeploymentEnv(pipelineParams.deployEnv, pipelineParams.envIndex, pipelineParams.gitGroup)                   
                            if(passiveCluster."${pipelineParams.gitGroup}_env" == "blue") {
                                envTag= 'SITB'
                                env.BLUE_GREEN = 'BLUE'
                                pipelineParams."bgTraffic" = 'SITB'
                                bgIndex = ''
                            }
                            else {
                                envTag= 'SITG'
                                env.BLUE_GREEN = 'GREEN'
                                pipelineParams."bgTraffic" = 'SITG'
                                bgIndex = 'bg'
                            }  
                        }                          
                        else{
                            println("bgTraffic in passive stage => ${bgTraffic}")
                            envTag = (bgTraffic == 'BLUE' || bgTraffic == 'blue') ? 'SITB' : 'SITG' 
                            pipelineParams."bgTraffic" = envTag
                        }
                        println("bgTraffic => ${pipelineParams.bgTraffic}")

                        // response = sh(script: "curl https://registry-sqa${envIndex}${bgIndex}-odc.vzwcorp.com/", returnStdout: true).trim()
                        // if(response.contains('Service Unavailable')){
                        //     echo "Registry Status for https://registry-sqa${envIndex}${bgIndex}-odc.vzwcorp.com/ is => ${response})"
                        //     sh "exit 1"
                        // }
                        // else{
                        //     echo "Registry Status for https://registry-sqa${envIndex}.tdc.vzwcorp.com/ is => UP"
                        // }
                    }
                }
            }
            stage("Dispatcher Deploy"){
                when {
                    expression { pipelineStages.service_dispatcher.toString() == 'true' }
                }
                steps{
                    script{
                        deployTasks.call(pipelineParams, pipelineParams.deployType)
                    }
                }
            }      
            stage("Fetch Deploy Service List"){
                when {
                    expression { pipelineStages.fetch_deployList.toString() == 'true' }
                }
                steps{
                    script{
                        def fetchedList = ec2PipelineHelper.generateDeployServicesList(pipelineParams.artifactoryHost,
                        pipelineParams.artifactoryRepo, 
                        pipelineParams.vsad,
                        pipelineParams.releaseId,
                        pipelineParams.deployEnv,
                        pipelineParams.deployType,
                        pipelineParams.full_deployment) //pipelineManager.call(pipelineParams, 'generateDeployServicesList')
                        steps.echo "fetchedList => ${fetchedList}"
                        if(fetchedList.toString().contains('Failed:')){
                            stageRun = false
                            status = 'FAILED'
                            error(fetchedList)
                        }
                        else if(fetchedList.toString().contains('Empty:')){
                            echo "${fetchedList}"
                            stageRun = false
                            status = 'SKIPPED'
                        }
                        else{
                            if(deployment_trigger.equalsIgnoreCase('adhoc')){
                                def tmp = paramJson.serviceList.split(",")
                                def filterList= [:]
                                def deployType
                                tmp.each{item ->
                                    //println((item.split("/"))[0].split(":")[1])
                                    echo "Item: ${item}"
                                    if(fetchedList.hashSvcToDeploy.toString().contains((item.split("/"))[1])){
                                        deployType=''
                                        deployType = fetchedList.hashSvcToDeploy.find{i ->
                                            i.toString().contains(item.split("/")[1])
                                            
                                        }
                                        steps.echo "deployType => ${deployType}"
                                        if(deployType){
                                            deployType = deployType.toString().split("=")[0]
                                            //(hashSvcToDeploy[value."DeploymentType"] == null ? "" : (hashSvcToDeploy[value."DeploymentType"] + ",")) + "${serviceName}:${path}/${imagetag}"
                                            filterList."${deployType}" = (filterList."${deployType}" == null ? "" : (filterList."${deployType}" + ",")) +  "${item}"
                                        }                                        
                                    }
                                }
                                //filterList = filterList.substring(0, filterList.length() -1)
                                echo "ServiceList to deploy: ${filterList}"
                                passiveDeploy."hashSvcToDeploy" = filterList 
                            }
                            else{
                                passiveDeploy = fetchedList
                            }
                            pipelineParams."passiveDeploy" = passiveDeploy
                        }
                    }
                }
            }
			stage("config_Deploy"){
				when{
					expression { params.Stages.config_Deploy.toString() == 'true' && stageRun == true}
				}
				steps{
					script{
						try {
							pipelineParams."config_Deploy" = 'true'
							echo "pipelineParams.config_Deploy => ${pipelineParams.config_Deploy}"
						}
                        
                            catch(Exception ex){
                            }
					}
				}
			}
            stage("Deploy-Services"){
                when{
                    expression { pipelineStages.service_Deploy.toString() == 'true' && stageRun == true}
                }
                steps{
                    script{
                        try{                                           
                            pipelineParams."rollBack" = 'yes'
                            echo "pipelineParams.deployType => ${pipelineParams.deployType}"
                            echo "passiveDeploy => ${passiveDeploy}"
                            if(passiveDeploy && passiveDeploy.dependentServiceToDeploy && pipelineParams.full_deployment.toString() == 'true'){                                
                                echo "====deploying dependent service first===="
                                    
                                //echo "services: ${serviceList.dependentServiceToDeploy}"
                                def tmpDeployments = [:]
                                passiveDeploy.hashDependentSvcToDeploy.each{key,value ->
                                    tmpDeployments["${key}"] = {
                                        pipelineParams."serviceList" = value
                                        pipelineParams."DeploymentType" = key
                                        deployTasks.call(pipelineParams, pipelineParams.deployType)
                                        echo "deployTasks success"
                                        status = 'SUCCESS'
                                        pipelineStages."update_DeployStatus" = 'true'

                                        
                                        echo "==== Updating deployment status for dependent modules ==="
                                        masterDeployStatusUrl = ec2DeployStatusTasks.call(pipelineParams, pipelineStages, envTag, activePassiveSync=false, updateTag)
                                    }
                                }
                                parallel tmpDeployments
                            }
                            if(passiveDeploy && passiveDeploy.serviceListToDeploy){
                                echo "====deploying modules===="
                                echo "hashSvcToDeploy: ${passiveDeploy.hashSvcToDeploy}"
                                def tmpDeployments = [:]
                                passiveDeploy.hashSvcToDeploy.each{key,value ->
                                    tmpDeployments["${key}"] = {
                                        pipelineParams."serviceList" = value
                                        pipelineParams."DeploymentType" = key
                                        pipelineParams."${key}_fileName" = "${pipelineParams.deployEnv}_${env.BUILD_NUMBER}_${key}.json" 
                                        deployTasks.call(pipelineParams, pipelineParams.deployType)
                                        echo "deployTasks success"
                                        //status = 'SUCCESS'
                                        pipelineStages."update_DeployStatus" = 'true'
                                        try{                          
                                            masterDeployStatusUrl = ec2DeployStatusTasks.call(pipelineParams, pipelineStages, envTag, activePassiveSync=false, updateTag) 
                                            if(masterDeployStatusUrl){
                                                echo "Update tag success"   
                                                echo "${masterDeployStatusUrl}"
                                                status = 'SUCCESS' 
                                                paramJson.message = ''
                                            }
                                            else{
                                                String skipMsg = "Empty response from deployment status. Either deployment is not happened or some error in getting deployment status"
                                                //paramJson.message = skipMsg
                                                echo "${skipMsg}"
                                                status = 'SKIPPED'
                                                //error(skipMsg)   
                                            }                             
                                        }
                                        catch(Exception ex){
                                            errMsg = "failed due to exception: " + ex.getMessage()
                                            echo errMsg
                                            //stageRun = false
                                            error(errMsg)                        
                                            //status = 'SKIPPED'
                                        }   
                                    }
                                }
                                parallel tmpDeployments
                            }                            
                        }
                        catch(Exception ex){
                            errMsg = "Error: Deploy service stage failed due to: ${ex.getMessage()}"
                            PipelineConstants pipelineConst = new PipelineConstants()
                            def recipient = pipelineConst.POS_DEPLOY_EMAILS
                            def jsonObj = ["Msg": "Deployment to ${pipelineParams.deployEnv} for release ${pipelineParams.releaseId} failed due to ${ex.getMessage()}"]
                            def title = "Failed: ${pipelineParams.gitGroup} Deployment on ${pipelineParams.deployEnv}"
                            def json = JsonOutput.toJson(jsonObj)
                            json = JsonOutput.prettyPrint(json)
                            mail(to: recipient, subject: title, body: json)
                            error("${errMsg}")
                            //stageRun = false
                            //
                        }                        
                    }
                }
            }
            stage("SRI Downstream"){
                when{
                   // expression { env.SRI_DEPLOY == 'TRUE'}
                   expression { pipelineParams.SRI_DEPLOY.toString() == 'TRUE' || env.SRI_DEPLOY == 'TRUE' }
                }
                steps{
                    script{
                        pipelineParams."SRI_DEPLOY" = env.SRI_DEPLOY
                        echo "SRI Deploy type ==> ${pipelineParams.SRI_DEPLOY}"
                        def artifcatNames = "${pipelineParams.serviceList}"
                        dEnv = "${pipelineParams.deployEnv}"
                        dEnv = "${dEnv}${pipelineParams.envIndex}"
                        if(artifcatNames.contains("|")) {
                            artifcatNames = artifcatNames.split("\\|")[1].trim()
                        }
                        artifcatNames = artifcatNames.toString()
                        echo "artifcatNames ==> ${artifcatNames}"
                        
                        PipelineConstants pipelineConst = new PipelineConstants()
                        def url = "${pipelineConst.HTNV_JOB}/buildWithParameters?ENV=${dEnv.toUpperCase()}&RELEASE=${pipelineParams.releaseId}&ARTIFACTORY_REPO=${pipelineParams.artifactoryRepo}&vsad=${pipelineParams.vsad.toLowerCase()}&ARTIFACT_NAME=${artifcatNames}"
                        withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'vcg2-jenkins', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]){
                        try {
                         response = sh(script: "curl -X POST -u \${USERNAME}:\${PASSWORD} \"$url\"", returnStdout: true)
                         echo "response => ${response}"
                        }
                        catch(Exception ex){
                            echo "Exception while Invoking HTNV Downstream JOB: ${ex.getMessage()}"
                         }

                           }
                        }
                }
           }
            stage("Update tag & Property"){
                when{
                    expression {stageRun == true}
                }
                steps{
                    script{
                        try{                          
                            masterDeployStatusUrl = ec2DeployStatusTasks.call(pipelineParams, pipelineStages, envTag, activePassiveSync=false, updateTag) 
                            if(masterDeployStatusUrl){
                                echo "Update tag success"   
                                echo "${masterDeployStatusUrl}"
                                status = 'SUCCESS' 
                                paramJson.message = ''
                            }
                            else{
                                String skipMsg = "Empty response from deployment status. Either deployment is not happened or some error in getting deployment status"
                                //paramJson.message = skipMsg
                                echo "${skipMsg}"
                                status = 'SKIPPED' 
                                error(skipMsg)
                            }                             
                        }
                        catch(Exception ex){
                            errMsg = "failed due to exception: " + ex.getMessage()
                            echo errMsg
                            //stageRun = false
                            error(errMsg)                        
                            //status = 'SKIPPED'
                        }                                     
                    }
                }
            }
            
        }
        post{
            success {
                script{
                    // paramJson.workflowStepStatus="SUCCESS"
                    // def json = JsonOutput.toJson(paramJson)
                    // json = JsonOutput.prettyPrint(json)
                    
                    // pipelineLog.call("cdAuditTrailLog", json)
                    manager.addShortText("Trigger=${deployment_trigger}, Stages=${stages}, Env=${pipelineParams.deployEnv}, Branch=${pipelineParams.releaseId}, Git_Group=${pipelineParams.gitGroup}, Full_Deployment=${full_deployment}, Skip_Active-Passive-Sync=${skipActivePassiveSync}")
                    cleanWs()
                }
            }
            failure {
                script{
                    // paramJson.workflowStepStatus= 'FAILED'
                    // paramJson.errorMessage = errMsg.toString()
                    // def json = JsonOutput.toJson(paramJson)
                    // json = JsonOutput.prettyPrint(json)
                    // pipelineLog.call("cdAuditTrailLog", json)
                    manager.addShortText("Trigger=${deployment_trigger}, Stages=${stages}, Env=${pipelineParams.deployEnv}, Branch=${pipelineParams.releaseId}, Git_Group=${pipelineParams.gitGroup}, Full_Deployment=${full_deployment}, Skip_Active-Passive-Sync=${skipActivePassiveSync}")
                    cleanWs()
                }
            }
        }
    }
}
