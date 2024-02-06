
IFS=' ' read -r -a SERVER_PORTS <<< "${SERVER_PORT}"

if [[ "${cm_src_dest}" != "-" ]]; then
  IFS=',' read -r -a CM_SRC_DEST_ITEMS <<< "${cm_src_dest}"
  for ITEM in "${CM_SRC_DEST_ITEMS[@]}"; do
    # Extract source and destination from the cm_src_dest item
    IFS=':' read -r tmpKey tmpVal <<< "${ITEM}"
    for PORT in "${SERVER_PORTS[@]}"; do
      TMPVAL_MODIFIED="${tmpVal//{{SERVER_PORT}}/${PORT}}"

      echo "Processing config for port ${PORT}: SRC: ${tmpKey}, DEST: ${TMPVAL_MODIFIED}"
      if [[ -d "${tmpKey}" ]]; then
        cp -r "${tmpKey}" "${TMPVAL_MODIFIED}"
      elif [[ -e "${tmpKey}" ]]; then
        cp "${tmpKey}" "${TMPVAL_MODIFIED}"
      else
        echo "SRC config file doesn't exist: ${tmpKey}"
        continue
      fi
      sed -i "s|{{SERVER_PORT}}|${PORT}|g" "${TMPVAL_MODIFIED}" || echo "Config file copy failed for port ${PORT}"
    done
  done
fi


application:
    - name: ecomm-akka-bbp-seed
      server_port: "2641,2642"
      limit_memory: "1g"
      instance_count: 2
      context: "ecomm_akka_bbp_seed"
      host: "{{Prefix}}" 
      registry: "{{EnvIndex}}_REGISTRY1"
      JAVA_OPTS:  "/app/bbportal/akka/ecomm_akka_bbp_seed/bin/ecomm_akka_bbp_seed -J-Xms512M -J-Xmx1g -J-server -J-javaagent:/app/infra/newrelic/newrelic.jar -Dnewrelic.config.app_name=EVBV-BBPORTAL-ecomm_akka_bbp_seed-GREEN-{{Region}}-AWSE -Dnewrelic.config.proxy_host=proxy.ebiz.verizon.com -Dnewrelic.config.proxy_port=80 -Dnewrelic.config.process_host.display_name={{hostIp}}.ebiz.verizon.com:bbportal:ecomm_akka_bbp_seed:{{server_port}}  -Dnewrelic.config.distributed_tracing.enabled=true $ADD_OPTS -DPROXY_HOST=proxy.ebiz.verizon.com -DPROXY_PORT=80 -Dorg.aspectj.tracing.factory=default -Djavax.net.ssl.keyStore=/app/rsa/eps/{{DisplayName}} -Djavax.net.ssl.keyStorePassword={{DisplayName2}} -Djavax.net.ssl.keyStoreType=JKS -Dakka.logger-startup-timeout=15s -DlogBuildInfoEnabled=Y -Dakka.cluster.auto-down-unreachable-after=10s -Dconfig.file=/app/bbportal/conf/bbportal.conf -Dlogback.configurationFile=/app/bbportal/conf/ecomm_akka_bbp_seed.{{server_port}}.logback.xml -Dakka.remote.netty.tcp.hostname={{hostIp}} -Dakka.remote.netty.tcp.port={{server_port}} -Dakka.cluster.seed-nodes.0=akka.tcp://application@{{hostIp}}:2641 -Dakka.cluster.seed-nodes.1=akka.tcp://application@{{hostIp}}:2642"
      cm_src_dest: "{{ServiceRoot}}/tmpapp/akka_config/roles/restart-akka/templates/logback.xml.j2:/app/bbportal/conf/ecomm_akka_bbp_seed.2641.logback.xml,{{ServiceRoot}}/tmpapp/akka_config/roles/restart-akka/templates/logback.xml.j2:/app/bbportal/conf/ecomm_akka_bbp_seed.2642.logback.xml,{{ServiceRoot}}/tmpapp/akka_config/roles/restart-akka/templates/logback.xml.j2:/app/bbportal/conf/ecomm_akka_bbp_seed.{{server_port}}.logback.xml{{ServiceRoot}}/tmpapp/akka_config/roles/restart-akka/templates/{{EnvIndex}}/bbportal.conf.j2:/app/bbportal/conf/bbportal.conf"





#!/bin/bash

# Assume SERVER_PORT is passed as an argument to this script
SERVER_PORT="$1"  # Example input could be "2461|2462"

# Check if SERVER_PORT contains a pipe '|'
if [[ "$SERVER_PORT" == *"|"* ]]; then
    echo "Multiple ports provided: $SERVER_PORT"
    # Split SERVER_PORT into an array of ports using the pipe as a delimiter
    IFS='|' read -r -a PORTS_ARRAY <<< "$SERVER_PORT"
    # Now you can iterate over ${PORTS_ARRAY[@]} to handle multiple ports
else
    echo "Single port provided: $SERVER_PORT"
    # Handle the single port case
fi

# Example of iterating over multiple ports if provided
for PORT in "${PORTS_ARRAY[@]}"; do
    echo "Processing port: $PORT"
    # Insert logic here to process each port
done

# Continue with script logic...

grep -q "^$ARTIFACT_INFO_APP_NAME:" $ARTIFACT_INFO_DEPLOY_DIR/$ARTIFACT_INFO_FILE_NAME && ${USE_SED} -i "s/^$ARTIFACT_INFO_APP_NAME:.*/$ARTIFACT_INFO_APP_NAME:${tmp_Artifact_url}/" $ARTIFACT_INFO_DEPLOY_DIR/$ARTIFACT_INFO_FILE_NAME || echo "$ARTIFACT_INFO_APP_NAME:${tmp_Artifact_url}" >> $ARTIFACT_INFO_DEPLOY_DIR/$ARTIFACT_INFO_FILE_NAME


[WARN] [01/19/2024 16:51:51.952] [application-akka.remote.default-remote-dispatcher-9] [akka.tcp://application@10.119.6.187:2641/system/endpointManager/reliableEndpointWriter-akka.tcp%3A%2F%2Fapplication%4010.119.6.187%3A2642-0] Association with remote system [akka.tcp://application@10.119.6.187:2642] has failed, address is now gated for [5000] ms. Reason: [Association failed with [akka.tcp://application@10.119.6.187:2642]] Caused by: [java.net.ConnectException: Connection refused: /10.119.6.187:2642]


+ sed -i 's/^ecomm_akka_bbp_seed:.*/ecomm_akka_bbp_seed:EVBV_1DBBPORTAL/vzw/evbv/release/24.02.100/ecomm-akka-bbp-seed/ecomm-akka-bbp-seed-f378cfd-310333-01152024164618.zip/' /app/bbportal/akka/ecomm_akka_bbp_seed/artifact.info
sed: -e expression #1, char 63: unknown option to `s'


fail=$(grep -lE "(APPLICATION FAILED TO START|Heap|OutOfMemoryError|Unable to access jarfile|Caused by:)" ${startup}/*/*.startup.out | grep -oE "($svcregex)" | sort -u | sed -n -e 'H;${x;s/\n/,/g;s/^,//;p;}')

fail=$(grep -rL "[WARN]" ${startup}/*/*.startup.out |
       grep -lE "(APPLICATION FAILED TO START|Heap|OutOfMemoryError|Unable to access jarfile|Caused by:)" |
       grep -oE "($svcregex)" |
       sort -u |
       sed -n -e 'H;${x;s/\n/,/g;s/^,//;p;}')

#!/usr/bin/bash
svc=$1
AppRoot=$2
startup=$3
svcregex=$(echo $svc | sed -e s/,/\|/g)

fail=$(grep -rL "[WARN]" ${startup}/*/*.startup.out | grep -lE "(APPLICATION FAILED TO START|Heap|OutOfMemoryError|Unable to access jarfile|Caused by:)" ${startup}/*/*.startup.out | grep -oE "($svcregex)" | sort -u | sed -n -e 'H;${x;s/\n/,/g;s/^,//;p;}')
echo "$fail"
if [ "$fail" != "" ]; then
  svc="$svc,$fail"
fi
echo "HealthFail:$fail"
pass=$(echo $svc | sed -e 's/,/\n/g' | sort | uniq -u | sed -n -e 'H;${x;s/\n/,/g;s/^,//;p;}')
echo "HealthPass:$pass"

exit 0

if [ "$cm_src_dest" != "-" ]; then
  for cm_item in ${cm_src_dest//,/ } ; do
    if [[ ${cm_item} =~ "{{server_port}}" ]]; then
      tmpKey=`echo ${cm_item} | cut -d ":" -f 1 | sed -i "s|{{SERVER_PORT}}|$SERVER_PORT|g"`
      tmpVal=`echo ${cm_item} | cut -d ":" -f 2 | sed -i "s|{{SERVER_PORT}}|$SERVER_PORT|g"`
      echo "SRC: ${tmpKey}, DEST: ${tmpVal}"
      if [ -d $tmpKey ]; then
        cp -r ${tmpKey} ${tmpVal}
      elif [ -e $tmpKey ]; then
        cp ${tmpKey} ${tmpVal}
        echo "sed -i \"s|{{APP_NAME}}|$APP_NAME|g;s|{{SERVER_PORT}}|$SERVER_PORT|g;s|{{ENV}}|$ENV|g;s|TIMESTAMP_VALUE|$TIMESTAMP_VALUE|g;s|{{LOGS_DIR}}|$LOGS_DIR|g\" ${tmpVal}"
        sed -i "s|{{APP_NAME}}|$APP_NAME|g;s|{{SERVER_PORT}}|$SERVER_PORT|g;s|{{ENV}}|$ENV|g;s|TIMESTAMP_VALUE|$TIMESTAMP_VALUE|g;s|{{LOGS_DIR}}|$LOGS_DIR|g" ${tmpVal}
      else
        echo "SRC config file doesn't exist"
      fi
    fi
    if [[ $? -ne 0 ]]; then
      echo "Config file copy failed";
    fi
  done
fi

cat ecomm_akka_bbp_seed.startup.out | grep -lE "(APPLICATION FAILED TO START|Heap|OutOfMemoryError|Unable to access jarfile|Caused by:)"
(standard input)

grep -rL "[WARN]" ecomm_akka_bbp_seed.startup.out | grep -lE "(APPLICATION FAILED TO START|Heap|OutOfMemoryError|Unable to access jarfile|Caused by:)" ecomm_akka_bbp_seed.startup.out | grep -oE "ecomm_akka_bbp_seed" | sort -u | sed -n -e 'H;${x;s/\n/,/g;s/^,//;p;}'



IFS=',' read -r -a cm_items <<< "$cm_src_dest"
for cm_item in "${cm_items[@]}"; do
  # Rest of the loop logic...
done
=====================

tmpKey=$(echo "${cm_item}" | cut -d ":" -f 1)
tmpVal=$(echo "${cm_item}" | cut -d ":" -f 2)
sed -i "s|{{SERVER_PORT}}|$SERVER_PORT|g" "$tmpKey" "$tmpVal"
=============================================

if [ "$cm_src_dest" != "-" ]; then
  for cm_item in ${cm_src_dest//,/ } ; do
    if [[ ${cm_item} =~ "{{SERVER_PORT_ACTUAL}}" ]]; then
      echo "cm_item:$cm_item"
      tmpKey=`echo ${cm_item} | cut -d ":" -f 1 | sed -e "s|{{SERVER_PORT_ACTUAL}}|$SERVER_PORT|g"`
      tmpVal=`echo ${cm_item} | cut -d ":" -f 2 | sed -e "s|{{SERVER_PORT_ACTUAL}}|$SERVER_PORT|g"`
      echo "SRC: ${tmpKey}, DEST: ${tmpVal}"
      if [ -d $tmpKey ]; then
        cp -r ${tmpKey} ${tmpVal}
      elif [ -e $tmpKey ]; then
        cp ${tmpKey} ${tmpVal}
        echo "sed -i \"s|{{APP_NAME}}|$APP_NAME|g;s|{{SERVER_PORT}}|$SERVER_PORT|g;s|{{ENV}}|$ENV|g;s|TIMESTAMP_VALUE|$TIMESTAMP_VALUE|g;s|{{LOGS_DIR}}|$LOGS_DIR|g\" ${tmpVal}"
        sed -i "s|{{APP_NAME}}|$APP_NAME|g;s|{{SERVER_PORT}}|$SERVER_PORT|g;s|{{ENV}}|$ENV|g;s|TIMESTAMP_VALUE|$TIMESTAMP_VALUE|g;s|{{LOGS_DIR}}|$LOGS_DIR|g" ${tmpVal
      else
        echo "SRC config file doesn't exist"
      fi
    elif [[ ${cm_item} != *"{{SERVER_PORT_ACTUAL}}"* ]]; then
      tmpKey=`echo ${cm_item} | cut -d ":" -f 1`
      tmpVal=`echo ${cm_item} | cut -d ":" -f 2`
      echo "SRC: ${tmpKey}, DEST: ${tmpVal}"
      if [ -d $tmpKey ]; then
        cp -r ${tmpKey} ${tmpVal}
      elif [ -e $tmpKey ]; then
        cp ${tmpKey} ${tmpVal}
        echo "sed -i \"s|{{APP_NAME}}|$APP_NAME|g;s|{{SERVER_PORT}}|$SERVER_PORT|g;s|{{ENV}}|$ENV|g;s|TIMESTAMP_VALUE|$TIMESTAMP_VALUE|g;s|{{LOGS_DIR}}|$LOGS_DIR|g\" ${tmpVal}"
        sed -i "s|{{APP_NAME}}|$APP_NAME|g;s|{{SERVER_PORT}}|$SERVER_PORT|g;s|{{ENV}}|$ENV|g;s|TIMESTAMP_VALUE|$TIMESTAMP_VALUE|g;s|{{LOGS_DIR}}|$LOGS_DIR|g" ${tmpVal}
      else
      echo "There is error in config"
      fi
    if [[ $? -ne 0 ]]; then
      echo "Config file copy failed";
    fi
  done
fi
=========================================================
if [ "$cm_src_dest" != "-" ]; then
  for cm_item in ${cm_src_dest//,/ } ; do
    if [[ "$cm_item" =~ "{{SERVER_PORT_ACTUAL}}" ]]; then
      echo "cm_item:$cm_item"
      tmpKey=$(echo "$cm_item" | cut -d ":" -f 1 | sed -e "s|{{SERVER_PORT_ACTUAL}}|$SERVER_PORT|g")
      tmpVal=$(echo "$cm_item" | cut -d ":" -f 2 | sed -e "s|{{SERVER_PORT_ACTUAL}}|$SERVER_PORT|g")
      echo "SRC: $tmpKey, DEST: $tmpVal"
      if [ -d "$tmpKey" ]; then
        cp -r "$tmpKey" "$tmpVal"
      elif [ -e "$tmpKey" ]; then
        cp "$tmpKey" "$tmpVal"
        sed -i "s|{{APP_NAME}}|$APP_NAME|g;s|{{SERVER_PORT}}|$SERVER_PORT|g;s|{{ENV}}|$ENV|g;s|TIMESTAMP_VALUE|$TIMESTAMP_VALUE|g;s|{{LOGS_DIR}}|$LOGS_DIR|g" "$tmpVal"
      else
        echo "SRC config file doesn't exist"
      fi
    elif [[ "$cm_item" != *"{SERVER_PORT_ACTUAL}"* ]]; then
      tmpKey=$(echo "$cm_item" | cut -d ":" -f 1)
      tmpVal=$(echo "$cm_item" | cut -d ":" -f 2)
      echo "SRC: $tmpKey, DEST: $tmpVal"
      if [ -d "$tmpKey" ]; then
        cp -r "$tmpKey" "$tmpVal"
      elif [ -e "$tmpKey" ]; then
        cp "$tmpKey" "$tmpVal"
      else
        echo "There is an error in config"
      fi
    fi  # This closes the if block
    if [[ $? -ne 0 ]]; then
      echo "Config file copy failed"
    fi
  done
fi
=================================================================

if [ "$cm_src_dest" != "-" ]; then
  for cm_item in ${cm_src_dest//,/ } ; do
    echo "Processing item: $cm_item"
    
    tmpKey=$(echo "$cm_item" | cut -d ":" -f 1 | sed "s|{{SERVER_PORT_ACTUAL}}|$SERVER_PORT|g")
    tmpVal=$(echo "$cm_item" | cut -d ":" -f 2 | sed "s|{{SERVER_PORT_ACTUAL}}|$SERVER_PORT|g")

    # Check if the placeholders were successfully replaced
    if [[ "$tmpKey" == *{{SERVER_PORT_ACTUAL}}* ]] || [[ "$tmpVal" == *{{SERVER_PORT_ACTUAL}}* ]]; then
      echo "Placeholder not replaced in $cm_item, skipping..."
      continue
    fi

    echo "SRC: $tmpKey, DEST: $tmpVal"
    if [ -d "$tmpKey" ]; then
      cp -r "$tmpKey" "$tmpVal"
    elif [ -e "$tmpKey" ]; then
      cp "$tmpKey" "$tmpVal"
      sed -i "s|{{APP_NAME}}|$APP_NAME|g;s|{{SERVER_PORT}}|$SERVER_PORT|g;s|{{ENV}}|$ENV|g;s|TIMESTAMP_VALUE|$TIMESTAMP_VALUE|g;s|{{LOGS_DIR}}|$LOGS_DIR|g" "$tmpVal"
    else
      echo "Source config file doesn't exist: $tmpKey"
    fi

    # Check the exit status of the last command
    if [[ $? -ne 0 ]]; then
      echo "Config file copy failed for item: $cm_item"
    fi
  done
fi
----------------------------------------======================\
if [ "$cm_src_dest" != "-" ]; then
  for cm_item in ${cm_src_dest//,/ } ; do
    if [[ "$cm_item" =~ "{{SERVER_PORT_ACTUAL}}" ]]; then
      echo "cm_item:$cm_item"
      tmpKey=$(echo "$cm_item" | cut -d ":" -f 1 | sed -e "s|{{SERVER_PORT_ACTUAL}}|$SERVER_PORT|g")
      tmpVal=$(echo "$cm_item" | cut -d ":" -f 2 | sed -e "s|{{SERVER_PORT_ACTUAL}}|$SERVER_PORT|g")
      echo "SRC: $tmpKey, DEST: $tmpVal"
      if [ -d "$tmpKey" ]; then
        cp -r "$tmpKey" "$tmpVal"
      elif [ -e "$tmpKey" ]; then
        cp "$tmpKey" "$tmpVal"
        sed -i "s|{{APP_NAME}}|$APP_NAME|g;s|{{SERVER_PORT}}|$SERVER_PORT|g;s|{{ENV}}|$ENV|g;s|TIMESTAMP_VALUE|$TIMESTAMP_VALUE|g;s|{{LOGS_DIR}}|$LOGS_DIR|g" "$tmpVal"
      else
        echo "SRC config file doesn't exist"
      fi
    elif [[ "$cm_item" != *"{{SERVER_PORT_ACTUAL}}"* ]]; then
      tmpKey=$(echo "$cm_item" | cut -d ":" -f 1)
      tmpVal=$(echo "$cm_item" | cut -d ":" -f 2)
      echo "SRC: $tmpKey, DEST: $tmpVal"
      if [ -d "$tmpKey" ]; then
        cp -r "$tmpKey" "$tmpVal"
      elif [ -e "$tmpKey" ]; then
        cp "$tmpKey" "$tmpVal"
      elif [ "$tmpVal" == *{{SERVER_PORT_ACTUAL}}* ]; then
      rm -rf "$tmpVal"
      else
        echo "There is an error in config"
      fi
    fi 
    if [[ $? -ne 0 ]]; then
      echo "Config file copy failed"
    fi
  done
fi
==================================================================

if [ "$cm_src_dest" != "-" ]; then
  for cm_item in ${cm_src_dest//,/ } ; do
    echo "Processing item: $cm_item"
    tmpKey=$(echo "$cm_item" | cut -d ":" -f 1 | sed "s|{{SERVER_PORT_ACTUAL}}|$SERVER_PORT|g")
    tmpVal=$(echo "$cm_item" | cut -d ":" -f 2 | sed "s|{{SERVER_PORT_ACTUAL}}|$SERVER_PORT|g")

    # Check for the existence of {{SERVER_PORT_ACTUAL}} in tmpVal
    if [[ "$tmpVal" == *{{SERVER_PORT_ACTUAL}}* ]]; then
      echo "Placeholder {{SERVER_PORT_ACTUAL}} found in tmpVal, removing file..."
      rm -rf "$tmpVal"
      continue
    fi

    echo "SRC: $tmpKey, DEST: $tmpVal"

    if [ -d "$tmpKey" ] || [ -e "$tmpKey" ]; then
      cp -r "$tmpKey" "$tmpVal"
      # Apply sed modifications if tmpKey is a file
      [ -e "$tmpKey" ] && sed -i "s|{{APP_NAME}}|$APP_NAME|g;s|{{SERVER_PORT}}|$SERVER_PORT|g;s|{{ENV}}|$ENV|g;s|TIMESTAMP_VALUE|$TIMESTAMP_VALUE|g;s|{{LOGS_DIR}}|$LOGS_DIR|g" "$tmpVal"
    else
      echo "Source config file doesn't exist: $tmpKey"
    fi

    if [[ $? -ne 0 ]]; then
      echo "Config file copy failed for item: $cm_item"
    fi
  done
fi
==============================================================================
if [ "$cm_src_dest" != "-" ]; then
  for cm_item in ${cm_src_dest//,/ } ; do
    echo "Processing item: $cm_item"
    tmpKey=$(echo "$cm_item" | cut -d ":" -f 1)
    tmpVal=$(echo "$cm_item" | cut -d ":" -f 2)

    # Replace placeholder only if SERVER_PORT is provided
    if [[ -n "$SERVER_PORT" ]]; then
      tmpKey=${tmpKey//\{\{SERVER_PORT_ACTUAL\}\}/$SERVER_PORT}
      tmpVal=${tmpVal//\{\{SERVER_PORT_ACTUAL\}\}/$SERVER_PORT}
    fi

    echo "SRC: $tmpKey, DEST: $tmpVal"

    # Check for the existence of {{SERVER_PORT_ACTUAL}} in tmpVal
    if [[ "$tmpVal" == *"\{\{SERVER_PORT_ACTUAL\}\}"* ]]; then
      echo "Placeholder {{SERVER_PORT_ACTUAL}} found in tmpVal, not creating file."
      continue
    fi

    if [ -d "$tmpKey" ] || [ -e "$tmpKey" ]; then
      cp -r "$tmpKey" "$tmpVal"
      # Apply sed modifications if tmpKey is a file
      if [ -e "$tmpKey" ]; then
        echo "Applying sed modifications to $tmpVal"
        sed -i "s#{{APP_NAME}}#$APP_NAME#g; s#{{SERVER_PORT}}#$SERVER_PORT#g; s#{{ENV}}#$ENV#g; s#TIMESTAMP_VALUE#$TIMESTAMP_VALUE#g; s#{{LOGS_DIR}}#$LOGS_DIR#g" "$tmpVal"
      fi
    else
      echo "Source config file doesn't exist: $tmpKey"
    fi

    # Check the exit status of the last command
    if [[ $? -ne 0 ]]; then
      echo "Config file copy failed for item: $cm_item"
    fi
  done
fi
==================================================================
if [ "$cm_src_dest" != "-" ]; then
  for cm_item in ${cm_src_dest//,/ } ; do
    echo "Processing item: $cm_item"

    # Escape the pipe character '|' in SERVER_PORT for use in sed
    escaped_server_port=$(echo "$SERVER_PORT" | sed 's/|/\\|/g')

    # Replace the placeholder with the escaped server port value
    tmpKey=$(echo "$cm_item" | cut -d ":" -f 1 | sed "s|{{SERVER_PORT_ACTUAL}}|$escaped_server_port|g")
    tmpVal=$(echo "$cm_item" | cut -d ":" -f 2 | sed "s|{{SERVER_PORT_ACTUAL}}|$escaped_server_port|g")

    echo "SRC: $tmpKey, DEST: $tmpVal"

    # Ensure that the file is not created if the placeholder remains
    if [[ "$tmpVal" == *{{SERVER_PORT_ACTUAL}}* ]]; then
      echo "Placeholder {{SERVER_PORT_ACTUAL}} found in tmpVal, not creating file."
      continue
    fi

    # Perform file operations
    if [ -d "$tmpKey" ]; then
      cp -r "$tmpKey" "$tmpVal"
    elif [ -e "$tmpKey" ]; then
      cp "$tmpKey" "$tmpVal"
      if [ -e "$tmpKey" ]; then
        echo "Applying sed modifications to $tmpVal"
        sed -i "s#{{APP_NAME}}#$APP_NAME#g; s#{{SERVER_PORT}}#$escaped_server_port#g; s#{{ENV}}#$ENV#g; s#TIMESTAMP_VALUE#$TIMESTAMP_VALUE#g; s#{{LOGS_DIR}}#$LOGS_DIR#g" "$tmpVal"
      fi
    else
      echo "Source config file doesn't exist: $tmpKey"
    fi

    # Check the exit status of the last command
    if [[ $? -ne 0 ]]; then
      echo "Config file copy failed for item: $cm_item"
    fi
  done
fi
===================================================
if [ "$cm_src_dest" != "-" ]; then
  for cm_item in ${cm_src_dest//,/ } ; do
    echo "Processing item: $cm_item"

    # Directly replacing placeholder without using sed to avoid issues with special characters
    tmpKey="${cm_item%%:*}"
    tmpVal="${cm_item##*:}"
    tmpKey="${tmpKey/\{\{SERVER_PORT_ACTUAL\}\}/$SERVER_PORT}"
    tmpVal="${tmpVal/\{\{SERVER_PORT_ACTUAL\}\}/$SERVER_PORT}"

    echo "SRC: $tmpKey, DEST: $tmpVal"

    # Check if tmpVal still contains the placeholder
    if [[ "$tmpVal" == *'{{SERVER_PORT_ACTUAL}}'* ]]; then
      echo "Placeholder still present in tmpVal after replacement, skipping creation of: $tmpVal"
      continue
    fi

    # Perform file operations
    if [ -d "$tmpKey" ] || [ -e "$tmpKey" ]; then
      cp -r "$tmpKey" "$tmpVal"
      # Apply sed modifications if tmpKey is a file
      if [ -f "$tmpKey" ]; then
        echo "Applying sed modifications to $tmpVal"
        sed -i "s#{{APP_NAME}}#${APP_NAME}#g; s#{{SERVER_PORT}}#${SERVER_PORT}#g; s#{{ENV}}#${ENV}#g; s#TIMESTAMP_VALUE#${TIMESTAMP_VALUE}#g; s#{{LOGS_DIR}}#${LOGS_DIR}#g" "$tmpVal"
      fi
    else
      echo "Source config file doesn't exist: $tmpKey"
    fi

    # Check the exit status of the last command
    if [[ $? -ne 0 ]]; then
      echo "Config file copy failed for item: $cm_item"
    fi
  done
fi
=================================================
if [ "$cm_src_dest" != "-" ]; then
  # Split SERVER_PORT into an array-like list of ports
  # This will be a single item array if there's no delimiter
  ports=(${SERVER_PORT//|/ })

  for cm_item in ${cm_src_dest//,/ } ; do
    echo "Processing item: $cm_item"

    tmpKey="${cm_item%%:*}"
    tmpVal="${cm_item##*:}"

    # Replace the placeholder with the actual server port value for each port
    for port in "${ports[@]}"; do
      # Use the PORT variable directly if it's a single integer
      # If PORT is not set, use the loop variable 'port'
      port_to_use=${PORT:-$port}

      modifiedKey="${tmpKey//\{\{SERVER_PORT_ACTUAL\}\}/$port_to_use}"
      modifiedVal="${tmpVal//\{\{SERVER_PORT_ACTUAL\}\}/$port_to_use}"

      echo "SRC: $modifiedKey, DEST: $modifiedVal"

      # Copy and modify the file for the current port
      if [ -d "$modifiedKey" ] || [ -e "$modifiedKey" ]; then
        cp -r "$modifiedKey" "$modifiedVal"
        if [ -f "$modifiedVal" ]; then
          # Apply the sed replacements for server port and app name within the file
          sed -i "s|{{APP_NAME}}|$APP_NAME|g; s|{{SERVER_PORT}}|$port_to_use|g; s|{{ENV}}|$ENV|g; s|TIMESTAMP_VALUE|$TIMESTAMP_VALUE|g; s|{{LOGS_DIR}}|$LOGS_DIR|g" "$modifiedVal"
        fi
      else
        echo "Source config file doesn't exist: $modifiedKey"
      fi

      # Check for errors after processing the current port
      if [[ $? -ne 0 ]]; then
        echo "Copy or modification failed for port $port_to_use: $cm_item"
        # If you want to stop the loop on failure, uncomment the next line
        # break
      fi
    done
  done
fi




=================================================
wget "${ARTIFACT_URL}" -q

if [[ $? -ne 0 ]]; then
  wget "${ARTIFACT_URL}" -q -O "${APP_NAME}.zip"
  
  if [[ $? -ne 0 ]]; then
    echo "Artifact: ${ARTIFACT_URL} download failed";
    exit 1;
  fi
  
  STAGED_FILE="${APP_NAME}.zip"
else
  FILENAME=$(basename "${ARTIFACT_URL}")
  STAGED_FILE="${FILENAME}"
fi

echo "Artifact: ${ARTIFACT_URL} downloaded";
echo "STAGED FILE: $(ls -l "${STAGED_FILE}")"
===================================================

echo "Deployment files found for: ${APP_NAME}"
wget ${ARTIFACT_URL} -P ${STAGE_DIR}/${APP_NAME} -q;
if [[ $? -ne 0 ]]; then
    echo "Artifact: ${ARTIFACT_URL} download failed";
    exit 1; 
fi
echo "Artifact: ${ARTIFACT_URL} download successful";

file_extensions=("zip" "tar" "war" "jar")

for file_ext in "${file_extensions[@]}"; do
    if ls -l "${STAGE_DIR}/${APP_NAME}/${APP_NAME}"*."${file_ext}" &>/dev/null; then
        echo "STAGED FILE: $(ls -l "${STAGE_DIR}/${APP_NAME}/${APP_NAME}"*."${file_ext}")"
        break
    else
        echo "No staged file found with .${file_ext} extension"
    fi
done

if [[ ${file_ext} == "${file_extensions[-1]}" ]]; then
    echo "No staged files found for any known extensions."
    exit 1
fi


-----=================================
echo "Backup created: $(ls -l $ARCHIVE_DIR/$APP_NAME/${APP_NAME}_$NOW.$file_ext)"
      if [[ $? -ne 0 ]]; then
      echo "STAGED FILE: $(ls -l $ARCHIVE_DIR/$APP_NAME/${APP_NAME}_$NOW.tar)"
      fi
=======================================================

file_path="$ARCHIVE_DIR/$APP_NAME/${APP_NAME}_$NOW.$file_ext"

if ls "$file_path" >/dev/null 2>&1; then
    echo "Backup created: $(ls -l "$file_path")"
else
    file_path="$ARCHIVE_DIR/$APP_NAME/${APP_NAME}_$NOW.tar"
    if ls "$file_path" >/dev/null 2>&1; then
        echo "STAGED FILE: $(ls -l "$file_path")"
    else
        echo "Backup file not found."
    fi
fi
====================================================
- name: Get content of JSON file using slurp
  slurp:
    src: "{{ ServiceRoot }}/{{ ARTIFACT_STATUS_FILE_NAME }}"
  register: json_file_content

- name: Print decoded content
  debug:
    msg: "{{ json_file_content['content'] | b64decode | replace('\n', '') }}"

- name: Parse the JSON content
  set_fact:
    json_content1: "{{ (json_file_content['content'] | b64decode | replace('\n', '')) | from_json }}"

- name: Fetch Previous Image Tag
  set_fact:
    previousServiceTag: "{{ json_content1.image_tag_before_rollback }}"

- name: Display the stored variables
  debug:
    msg: "previousServiceTag: {{ previousServiceTag }}"

- name: Parse the Second JSON content (This might not be necessary if it's the same content as before)
  set_fact:
    json_content2: "{{ (json_file_content['content'] | b64decode | replace('\n', '')) | from_json }}"

- name: Fetch Current Image Tag
  set_fact:
    currentServiceTag: "{{ json_content2.image_tag_after_rollback }}"

- name: Display the stored variables
  debug:
    msg: "currentServiceTag: {{ currentServiceTag }}"

- name: Extract commit id of previousServiceTag
  set_fact:
    commitid_brb: "{{ previousServiceTag | regex_search('^(?:[^.]*\\.){2}...(.{7})', '\\1') | default('') }}"

- name: Extract commit id of currentServiceTag
  set_fact:
    commitid_arb: "{{ currentServiceTag | regex_search('^(?:[^.]*\\.){2}...(.{7})', '\\1') | default('') }}"

=======================================================================================================

jq '. += input' existing.json newData.json > temp.json && mv temp.json existing.json

jq -n \
  --arg e "$e" \
  --arg f "$f" \
  --arg g "$g" \
  --arg h "$h" \
  '{a: $e, b: $f, c: $g, d: $h}' > myfile.json

}


echo "{
\"a\":\"$e\",
\"b\":\"$f\",
\"c\":\"$g\",
\"d\":\"$h\"
}" > myfile.json



