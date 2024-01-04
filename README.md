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
                                dEnv =  deployEnv.toLowerCase()
                                envIndex= dEnv.trim().reverse().take(1)
                                dEnv = dEnv.substring(0, dEnv.length() -1);
                                deployConfigProperty = ec2PipelineInit.getDeploymentConfigs(gitGroup, deployEnv, deployType, '', pCategory)
                                
                                echo "pipelineStages => ${pipelineStages}"
                                echo "pipelineParams ==> ${pipelineParams}"
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
                        /*catch(Exception ex){
                            errMsg = "failed due to exception: " + ex.getMessage()
                            error errMsg
                        }*/
                            }
