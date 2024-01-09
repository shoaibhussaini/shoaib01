try{
						def params = [:]
						def gitGroup = ""
						def AUTOSCALE_GROUP1 = ""
						params.GITLAB_GROUP = gitGroup
						if(INFRA_TYPE && (INFRA_TYPE == "onprem" || INFRA_TYPE == "on-prem")){
							zip zipFile: 'pipelinesrc_src.zip', archive: false, dir: './EC2'
							withCredentials([usernameColonPassword(credentialsId: 'SVC-unified-cicd', variable: 'USERPASS')]) {
							sh "curl -s -u $USERPASS -T ./pipelinesrc_src.zip https://oneartifactoryci.verizon.com/artifactory/I2IV_SRE/pipelinesrc_src_${PlaybookBranch}.zip"
							}
						}
						cleanWs()
						Vsad = "${VSAD}"
						SvcNames = ""						
						checkout([$class: 'GitSCM', branches: [[name: "${GIT_URL_CONFIG_YAML}".split('\\|')[1]]], 										userRemoteConfigs: [
							[credentialsId: 'SVC-unified-cicd' ,url: 'https://' + "${GIT_URL_CONFIG_YAML}".split('\\|')[0]]]]
						)
						println("**** Git Files used for this deployment ***** ")
						println("Git repository :"+"${GIT_URL_CONFIG_YAML}".split('\\|')[0])
						println("Git branch :"+"${GIT_URL_CONFIG_YAML}".split('\\|')[1])
						println(("${GIT_URL_CONFIG_YAML}".split('\\|').length > 2 ? "${GIT_URL_CONFIG_YAML}".split('\\|')[2] : VSAD).toLowerCase() + '-config.yml')
						if(INFRA_TYPE && (INFRA_TYPE == "onprem" || INFRA_TYPE == "on-prem")){
							zip zipFile: 'hivv_src.zip', archive: false, dir: './config'
							withCredentials([usernameColonPassword(credentialsId: 'SVC-unified-cicd', variable: 'USERPASS')]) {
								config_branch = "${GIT_URL_CONFIG_YAML}".split('\\|')[1]
								sh "curl -s -u $USERPASS -T ./hivv_src.zip https://oneartifactoryci.verizon.com/artifactory/I2IV_SRE/hivv_src_${config_branch}.zip"
							}
						}
						
						
						/*if(VSAD == "G1XV" && EnvIndex == "PRODE") {
							AUTOSCALE_GROUP = "vz-App-G1XV-ONESEARCH-prod2-AutoScalingGroup"
						}
						if(VSAD == "G1XV" && EnvIndex == "PRODW") {
							AUTOSCALE_GROUP = "vz-App-G1XV-ONESEARCH-prod2-AutoScalingGroup"
						}*/
						if(AUTOSCALE_GROUP && AUTOSCALE_GROUP != "" && AUTOSCALE_GROUP != "null") {
							AUTOSCALE_GROUP1 = AUTOSCALE_GROUP
						}
						println("AUTOSCALE_GROUP:"+AUTOSCALE_GROUP)
						println("AUTOSCALE_GROUP1:"+AUTOSCALE_GROUP1)						
						
						if(AUTOSCALE_GROUP1 && AUTOSCALE_GROUP1 != "" && AUTOSCALE_GROUP1 != "null") {
							println("***AutoScale group present. finding the Instance IPs..")
							def paramsAutoscale = [:]
							paramsAutoscale."ASGEXPORTNAME" = AUTOSCALE_GROUP1
							paramsAutoscale."USER_ID" = "sunddi6"
							paramsAutoscale."RELEASE_BRANCH" = Branch
							paramsAutoscale."REGION" = "${REGION}"
							paramsAutoscale."ACTION" = "AUTOSCALE"
							paramsAutoscale."GITLAB_GROUP" = "${GITLAB_GROUP}"
							paramsAutoscale."ARTIFACT_STATUS_FILE_NAME" = "host-${ARTIFACT_STATUS_FILE_NAME}"
							paramsAutoscale."PROXY" = "http://vzproxy.verizon.com:80"
							withCredentials([[
								$class: 'UsernamePasswordMultiBinding', credentialsId: gitCreds,
								usernameVariable: 'GIT_USR', passwordVariable: 'GIT_PWD'
								]]) {
								paramsAutoscale."GITLAB_CREDS" = "${GIT_USR}:${GIT_PWD}"
							}
							step([$class: AnsiblePlugin, ansibleVersion: AnsibleVersion, 
								gitUrl: "${GIT_URL}".replace("https://","git@").replace(".com/",".com:"), 
								parameter: JsonOutput.toJson(paramsAutoscale), playbook: "EC2/ec2-config-deploy.yml", 
								os: OS, gitBranch: PlaybookBranch, 
								credentialsId: AnsibleCreds, vsad: 'I2IV', 
								serverEnv: [awsNpUserName: AWSUser, includeUser: false], 
								inv: [serverIps: ServerIP_Autoscale], vaultCredentialsId: '', 
								extraArgs: ''])
							
							sh "curl -L https://${paramsAutoscale.GITLAB_CREDS}@oneartifactoryci.verizon.com/artifactory/I2IV_SRE/${GITLAB_GROUP}/ec2/host-${ARTIFACT_STATUS_FILE_NAME} > host-${ARTIFACT_STATUS_FILE_NAME}"

							def tmpStatus = readProperties file: "./host-${ARTIFACT_STATUS_FILE_NAME}"
							def tempIps = ""
							tmpStatus.each{key,value ->							
								tempIps += "," + key
							}
							tempIps = tempIps.substring(1)
							sh "sed -i -e 's/AUTOSCALE_GROUP/${tempIps}/g' ./config/" + ("${GIT_URL_CONFIG_YAML}".split('\\|').length > 2 ? "${GIT_URL_CONFIG_YAML}".split('\\|')[2] : VSAD).toLowerCase() + '-config.yml'
						}
						

						def configyamls = readYaml interpolate: false, file: './config/' + ("${GIT_URL_CONFIG_YAML}".split('\\|').length > 2 ? "${GIT_URL_CONFIG_YAML}".split('\\|')[2] : VSAD).toLowerCase() + '-config.yml'
						EnviIndex3 = ""
						EnviIndex2 = ""
						DisplayName = ""
						DisplayName2 = ""
						DeployEnv = "${EnvIndex}"
						Region=""
						if(configyamls.global != null && configyamls.global.Prefix != null && configyamls.global.Prefix."${EnvIndex}" != null) {
							EnviIndex = configyamls.global.Prefix."${EnvIndex}"
						}
						if(configyamls.global != null && configyamls.global.Prefix2 != null && configyamls.global.Prefix2."${EnvIndex}" != null) {
							EnviIndex2 = configyamls.global.Prefix2."${EnvIndex}"
						}
						if(configyamls.global != null && configyamls.global.Prefix3 != null && configyamls.global.Prefix3."${EnvIndex}" != null) {
							EnviIndex3 = configyamls.global.Prefix3."${EnvIndex}"
						}
						if(configyamls.global != null && configyamls.global.Prefix_Env != null && configyamls.global.Prefix_Env."${EnvIndex}" != null) {
							EnviIndex_Prefix = configyamls.global.Prefix_Env."${EnvIndex}"
						}
						if(configyamls.global != null && configyamls.global.DisplayName != null && configyamls.global.DisplayName."${EnvIndex}" != null) {
							DisplayName = configyamls.global.DisplayName."${EnvIndex}"
							println("DisplayName:- ${DisplayName}")
						}
						if(configyamls.global != null && configyamls.global.DisplayName2 != null && configyamls.global.DisplayName2."${EnvIndex}" != null) {
							DisplayName2 = configyamls.global.DisplayName2."${EnvIndex}"
							println("DisplayName2:- ${DisplayName2}")
						}
						if(configyamls.global != null && configyamls.global.Region != null && configyamls.global.Region."${EnvIndex}" != null) {
							Region = configyamls.global.Region."${EnvIndex}"
						}
						if(configyamls.global != null && configyamls.global.Prefix_user != null) {
							Prefix_user = configyamls.global.Prefix_user
						}
						if(configyamls.global != null && configyamls.global.Checkout_location != null) {
							Checkout_location = configyamls.global.Checkout_location
						}
						if(configyamls.global != null && configyamls.global.Ansible_version != null) {
							Ansible_version = configyamls.global.Ansible_version

						}
						if(configyamls.global != null && configyamls.global.Prefix_host != null) {
							Prefix_host = configyamls.global.Prefix_host
						}
						if(configyamls.global != null && configyamls.global.Prefix_Server_Number != null) {
							Prefix_Server_Number = configyamls.global.Prefix_Server_Number
						}
						if(configyamls.global != null && configyamls.global.AWS_USER != null) {
							AWS_USER = configyamls.global.AWS_USER
						}
						if(configyamls.global != null && configyamls.global.CONFIGMAP != null && configyamls.global.Checkout_location == "artifactory") {
							config_repo = configyamls.global.CONFIGMAP.SRC_REPO
							config_branch = configyamls.global.CONFIGMAP.SRC_REPO_BRANCH
							println("config_repo: ${config_repo}, config_branch: ${config_branch}")
							checkout([
								$class: 'GitSCM', 
								branches: [[name: "${config_branch}"]],
								userRemoteConfigs: [[
									credentialsId: 'SVC-unified-cicd' ,
									url: 'https://gitlab.verizon.com/' + "${config_repo}"
								]]
							])
						    zip zipFile: 'app_config.zip', archive: false
						    withCredentials([usernameColonPassword(credentialsId: 'SVC-unified-cicd', variable: 'USERPASS')]) {
							sh "curl -s -u $USERPASS -T ./app_config.zip https://oneartifactoryci.verizon.com/artifactory/I2IV_SRE/app_config.zip"
						    }
						}
						if("${ServiceNames}".contains("|")) {
							def registryMap = [:]
							def svcMap = [:]
							def tmpConfig = ""
							for (def str : "${ServiceNames}".split('\\|')[1].split(",")) {
								if(str.toUpperCase() == 'ALL') {
									configyamls.application.each { tmp_application ->
										tmpConfig = tmp_application
										if(tmpConfig && tmpConfig.host) {								
											tmpConfig.host = tmpConfig.host.replace('{{Prefix}}', EnviIndex).replace('{{Prefix2}}', EnviIndex2).replace('{{Prefix3}}', EnviIndex3)
											def index = 0
											for (def hst : tmpConfig.host.split(",")) {
												if(hst == "${ServiceNames}".split('\\|')[0]) {
													svcMap[hst] = (svcMap[hst] == null ? "" : svcMap[hst]) + (svcMap[hst] == null ? "" : "|") + tmpConfig.name
													def registry = "-"
													if(tmpConfig.registry) {
														registry = tmpConfig.registry.split(",")[index].replace('{{Prefix}}', EnviIndex).replace('{{Prefix2}}', EnviIndex2).replace('{{Prefix3}}', EnviIndex3).replace('{{EnvIndex}}', "${EnvIndex}")
													}
													if(configyamls.ALB && configyamls.ALB."${registry}") {
														registryMap[hst] = configyamls.ALB."${registry}".replace('{{Prefix}}', EnviIndex).replace('{{Prefix2}}', EnviIndex2).replace('{{Prefix3}}', EnviIndex3).replace('{{EnvIndex}}', "${EnvIndex}")
													} else {
														registryMap[hst] = "-"
													}												
													
													index = index + 1
												}
											}
										}
									}
								} else {									
									tmpConfig = configyamls.application.find{ obj -> obj.name == str.split(":")[0] }
									if(tmpConfig && tmpConfig.host) {								
										tmpConfig.host = tmpConfig.host.replace('{{Prefix}}', EnviIndex).replace('{{Prefix2}}', EnviIndex2).replace('{{Prefix3}}', EnviIndex3)
										def index = 0
										for (def hst : tmpConfig.host.split(",")) {
											def registry = "-"
											if(tmpConfig.registry) {
												registry = tmpConfig.registry.split(",")[index].replace('{{Prefix}}', EnviIndex).replace('{{Prefix2}}', EnviIndex2).replace('{{Prefix3}}', EnviIndex3).replace('{{EnvIndex}}', "${EnvIndex}")
											}
											if(configyamls.ALB && configyamls.ALB."${registry}") {
												registryMap[hst] = configyamls.ALB."${registry}".replace('{{Prefix}}', EnviIndex).replace('{{Prefix2}}', EnviIndex2).replace('{{Prefix3}}', EnviIndex3).replace('{{EnvIndex}}', "${EnvIndex}")
											} else {
												registryMap[hst] = "-"
											}
											index = index + 1								
										}
									}
								}
							}
							if("${ServiceNames}".split('\\|')[1].split(",")[0].toUpperCase() == 'ALL') {
								SvcNames = "${ServiceNames}".split('\\|')[0] + "," + svcMap["${ServiceNames}".split('\\|')[0]] + "," + registryMap["${ServiceNames}".split('\\|')[0]]
							} else {
								SvcNames = "${ServiceNames}".replace("|","||").replace(",","|").replace("||",",") + "," + registryMap["${ServiceNames}".split('\\|')[0]]
							}							
						}
						else {
							def svcMap = [:]
							def registryMap = [:]
							def tmpConfig = ""
							for (def str : "${ServiceNames}".split(",")) {
								if(str.toUpperCase() == 'ALL') {
									configyamls.application.each { tmp_application ->
										tmpConfig = tmp_application
										if(tmpConfig && tmpConfig.host) {								
											tmpConfig.host = tmpConfig.host.replace('{{Prefix}}', EnviIndex).replace('{{Prefix2}}', EnviIndex2).replace('{{Prefix3}}', EnviIndex3)
											def index = 0
											for (def hst : tmpConfig.host.split(",")) {									
												svcMap[hst] = (svcMap[hst] == null ? "" : svcMap[hst]) + "|" + tmpConfig.name
												def registry = "-"
												if(tmpConfig.registry) {
													registry = tmpConfig.registry.split(",")[index].replace('{{Prefix}}', EnviIndex).replace('{{Prefix2}}', EnviIndex2).replace('{{Prefix3}}', EnviIndex3).replace('{{EnvIndex}}', "${EnvIndex}")
												}
												if(configyamls.ALB && configyamls.ALB."${registry}") {
													registryMap[hst] = configyamls.ALB."${registry}".replace('{{Prefix}}', EnviIndex).replace('{{Prefix2}}', EnviIndex2).replace('{{Prefix3}}', EnviIndex3).replace('{{EnvIndex}}', "${EnvIndex}")
												} else {
													registryMap[hst] = "-"
												}
												index = index + 1								
											}
										}
									}
								} else {
									tmpConfig = configyamls.application.find{ obj -> obj.name == str.split(":")[0] }
									if(tmpConfig && tmpConfig.host) {								
										tmpConfig.host = tmpConfig.host.replace('{{Prefix}}', EnviIndex).replace('{{Prefix2}}', EnviIndex2).replace('{{Prefix3}}', EnviIndex3)
										def index = 0
										for (def hst : tmpConfig.host.split(",")) {									
											svcMap[hst] = (svcMap[hst] == null ? "" : svcMap[hst]) + "|" + str
											def registry = "-"
											if(tmpConfig.registry) {
												registry = tmpConfig.registry.split(",")[index].replace('{{Prefix}}', EnviIndex).replace('{{Prefix2}}', EnviIndex2).replace('{{Prefix3}}', EnviIndex3).replace('{{EnvIndex}}', "${EnvIndex}")
											}
											if(configyamls.ALB && configyamls.ALB."${registry}") {
												registryMap[hst] = configyamls.ALB."${registry}".replace('{{Prefix}}', EnviIndex).replace('{{Prefix2}}', EnviIndex2).replace('{{Prefix3}}', EnviIndex3).replace('{{EnvIndex}}', "${EnvIndex}")
											} else {
												registryMap[hst] = "-"
											}
											index = index + 1								
										}
									}
								}
							}
							println("registryMap:"+registryMap)
							svcMap.each{key,value ->							
								SvcNames += "||" + key + "," + value.substring(1) + "," + registryMap[key]
							}
							if(SvcNames != "") {
								SvcNames = SvcNames.substring(2)
							}
						}
												
						println('SvcNames:'+SvcNames)
						BuildNO = "${BUILD_NUMBER}"
						Branch = "${RELEASE_BRANCH}"
						ArtifactoryURL = ARTIFACT_URL
						wrap([$class: 'BuildUser']) {
							EmailAddress = EmailAddress + "," + env.BUILD_USER_EMAIL
							println 'EmailAddress:' + EmailAddress
						}
						validateParams("VSAD", Vsad)
						validateParams("ServiceNames", SvcNames)
						validateParams("RELEASE_BRANCH", Branch)
						validateParams("ARTIFACT_URL", ArtifactoryURL)
						validateParams("ANSIBLE_PLUGIN", AnsiblePlugin)
						validateParams("ANSIBLE_VERSION", AnsibleVersion)
						validateParams("GITLAB_GROUP", "${GITLAB_GROUP}")
						validateParams("ACTION", "${ACTION}")
						validateParams("GIT_URL_CONFIG_YAML", "${GIT_URL_CONFIG_YAML}")
						validateParams("configDeploy", "${configDeploy}")
					}
