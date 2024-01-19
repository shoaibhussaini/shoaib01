
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


cm_src_dest: "{{ServiceRoot}}/tmpapp/akka_config/roles/restart-akka/templates/logback.xml.j2:/app/bbportal/conf/ecomm_akka_bbp_seed.2641.logback.xml,{{ServiceRoot}}/tmpapp/akka_config/roles/restart-akka/templates/logback.xml.j2:/app/bbportal/conf/ecomm_akka_bbp_seed.2642.logback.xml,{{ServiceRoot}}/tmpapp/akka_config/roles/restart-akka/templates/logback.xml.j2:/app/bbportal/conf/ecomm_akka_bbp_seed.{{server_port}}.logback.xml{{ServiceRoot}}/tmpapp/akka_config/roles/restart-akka/templates/{{EnvIndex}}/bbportal.conf.j2:/app/bbportal/conf/bbportal.conf"
