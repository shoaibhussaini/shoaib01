if [ ${RESTART} == 'yes' ]; then  
  SERVER_PORT=`echo ${SERVER_PORT} | sed -e s/\|/,/g`
  PORT_COUNT=`echo ${SERVER_PORT}|awk -F',' '{print NF}'`
  echo "Starting ${INSTANCE_NAME}......"
  if [[ $PORT_COUNT -eq 1 ]]; 
  then
    if [[ "$INSTANCE_START_INDEX" && -n "${INSTANCE_START_INDEX}" ]]; then
      for i in `seq $INSTANCE_START_INDEX $INSTANCE_COUNT` ; do
        j=${i}
        $SCRIPTS_DIR/startinstance.sh ${INSTANCE_NAME} ${j} ${SCRIPTS_DIR} ${SERVER_PORT} $INSTANCE_COUNT $ENV $MEM $VSAD $AppRoot 'true' ${curr_work_dir}
        sleep 2
      done
    else 
        for i in `seq 1 $INSTANCE_COUNT` ; do
        j=${i}
        if [[ $j -lt 10 ]];then
          j="0$j"
        fi
        $SCRIPTS_DIR/startinstance.sh ${INSTANCE_NAME} ${j} ${SCRIPTS_DIR} ${SERVER_PORT} $INSTANCE_COUNT $ENV $MEM $VSAD $AppRoot 'true' ${curr_work_dir}
        sleep 2
      done
    fi
  elif [[ $PORT_COUNT -gt 1 &&  $PORT_COUNT -eq $INSTANCE_COUNT ]]; 
  then
    if [[ "$INSTANCE_START_INDEX" && -n "${INSTANCE_START_INDEX}" ]]; then
        i=$INSTANCE_START_INDEX
        for port_item in ${SERVER_PORT//,/ } ; do
          j=${i}
          $SCRIPTS_DIR/startinstance.sh ${INSTANCE_NAME} ${j} ${SCRIPTS_DIR} ${port_item} $INSTANCE_COUNT $ENV $MEM $VSAD $AppRoot 'false' ${curr_work_dir}
          sleep 2
          i=$(expr ${i} + 1)
        done
    else 
      i=1
        for port_item in ${SERVER_PORT//,/ } ; do
          j=${i}
          if [[ $j -lt 10 ]];then
            j="0$j"
          fi
          $SCRIPTS_DIR/startinstance.sh ${INSTANCE_NAME} ${j} ${SCRIPTS_DIR} ${port_item} $INSTANCE_COUNT $ENV $MEM $VSAD $AppRoot 'false' ${curr_work_dir}
          sleep 2
          i=$(expr ${i} + 1)
        done
    fi
  else
    echo "Server port and instance count mismatch. Please validate"
    exit 1
  fi
fi
