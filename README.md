ALB:
    DIT1_REGISTRY1: "http://{{Prefix}}:1111/"

global:
    ENV:
      JAVA_HOME: "/app/jdk/jdk1.8.0_73"
      LOGS_DIR: "/log/shopservices"
    JAVA_OPTS: "{{JAVA_HOME}}/bin/java {{JAVA_OPTS_CONFIG}}"
    CONFIGMAP:
      SRC_REPO: "TYPEB_MW/shop_services_rolling"
      SRC_REPO_BRANCH: "master"

    Prefix:
      DIT1: qa6-uservices.ebiz.verizon.com
      #add the above two lines in the mediator, pass SIT1B/SIT1G in the env
    Prefix_Env:
      DIT1: QA6
      
application:
    - name: vzw-onesearch
      server_port: 8085
      limit_memory: "512M"
      instance_count: 2
      context: vzw-onesearch
      host: "{{Prefix}}"
      registry: "{{EnvIndex}}_REGISTRY1"
      JAVA_OPTS: "-Xms256M -Xmx512M -server -javaagent:/app/infra/newrelic/newrelic.jar -Xloggc:/log/shopservices/OneSearch_integrated/OneSearch_integrated.gc.log -javaagent:/app/prevoty/prevoty-agent.jar -Dprevoty.config=/app/prevoty/23868_ESP_RASPModConf.json -Dprevoty.log.config=/app/prevoty/23868_ESP_RASPLogConf.json -Dprevoty.raspjarpath=/app/prevoty/prevoty-rasp.jar -Dnewrelic.config.proxy_host=proxy.ebiz.verizon.com -Dnewrelic.config.proxy_port=80 -Dnewrelic.config.app_name=CV9V-SHOPSERVICES-OneSearch_integrated-GREEN-{{EnvIndex}}-AWSE -Dserver.port={{server_port}} -DartifactDeployed=/app/latest/vzw-onesearch/vzw-onesearch*.jar -Dlogging.file=/log/shopservices/OneSearch_integrated/OneSearch_integrated-{{EnvIndex}}-app.log -Dspring.config.location=/app/shopservices/conf/application-search-{{EnvIndex}}.properties -jar /app/vzw/latest/vzw-onesearch/vzw-onesearch*.jar"
      cm_src_dest: "{{ServiceRoot}}/tmpapp/roles/restart-uservice1/templates/application.search.{{EnvIndex}}.properties.j2:/app/shopservices/conf/application-search-{{EnvIndex}}.properties"
