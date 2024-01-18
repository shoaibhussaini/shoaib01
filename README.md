- name: Calling {{ ACTION }}
  shell: "{{ AppRoot + '/upscripts/' + ACTION }}.sh {{ RELEASE_BRANCH }} {{ ServiceName }} {{ ARTIFACT_URL + '/' + RELEASE_BRANCH + '/' + deploy_item.split(\":\")[1] }} {{ RegistryURL }} {{\"'\"+(ServiceConfig.server_port | replace(',','|')) + \"'\" if  '|' in  (ServiceConfig.server_port | replace(',','|')) else ServiceConfig.server_port -1 }} {{ ServiceConfig.instance_count }} {{ EnvIndex }} {{ ServiceConfig.limit_memory }} {{ VSAD }} {{ AppRoot }} {{ (\"'\"+JavaOpts+\"'\") | replace('{{JAVA_OPTS_CONFIG}}',ServiceConfig.JAVA_OPTS) | replace('{{Prefix_Server_Number}}', Prefix_Server_Number) | replace('{{host_CONFIG_Prefix}}', Prefix_host) | replace('{{limit_memory}}',ServiceConfig.limit_memory) | replace('{{ServiceName}}',ServiceName) | replace('{{DisplayName}}',DisplayName) | replace('{{DisplayName2}}',DisplayName2) | replace('{{Region}}',Region) | replace('{{SERVICENAME}}',(ServiceName | upper)) | replace(' ','|') | replace('{{ROOT_DIR}}',AppRoot) | replace('{{ARTIFACT_VERSION}}',deploy_item.split(\":\")[1]) }} {{ (\"'\"+((AppEnv | combine (ServiceConfig.ENV if ServiceConfig.ENV is defined and ServiceConfig.ENV !='' else AppEnv)).items() | map('join', ':') | join(',') | replace(' ', '|'))+\"'\") }} {{ ServiceNames.split(\",\")[0] }} {{ AppRestart if AppRestart is defined and AppRestart == RESTART else RESTART }} {{ DEPLOY_ENV }} {{ (\"'\"+(ServiceConfig.cm_src_dest | replace( '{{ServerPort}}',ServiceConfig.server_port) if ServiceConfig.cm_src_dest is defined and ServiceConfig.cm_src_dest !='' else '-')+\"'\") }}"
  vars:
    ServiceName: "{{ deploy_item.split(\":\")[0] }}"
    config_service_query: "application[?name=='{{ ServiceName }}']"
    ServiceConfig: "{{ ((ServiceConfigFile.stdout | from_yaml) | json_query(config_service_query)) | first | replace('{{vsad}}', VSAD | lower) | replace('{{VSAD}}', VSAD | upper) | replace('{{EnvIndex}}', EnvIndex | lower) | replace('{{ENVINDEX}}', EnvIndex | upper) | replace('{{ServiceRoot}}', ServiceRoot) | replace('{{TIMESTAMP_VALUE}}', TIMESTAMP_VALUE) | replace('{{RELEASE}}', RELEASE_BRANCH.split('/').1)}}"
  loop_control:
    loop_var: deploy_item
  register: deploy_item_loop_var1
  with_items:
    - "{{ ServiceNamesAllArtifact.split(\"|\") }}"
  when: ACTION == 'deploy'
