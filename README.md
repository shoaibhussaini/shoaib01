#!/bin/bash
VERSION=$1
APP_NAME=$2
ARTIFACT_URL=$3
REGISTRY=$4
SERVER_PORT=$5
INSTANCE_COUNT=$6
ENV=$7
MEM=$8
VSAD=${9}
AppRoot=${10}
JavaOpts=${11}
AppEnv=${12}
HOST_NAME=${13}
RESTART=${14}
DEPLOY_ENV=${15}
cm_src_dest=${16}

BG_VALUE=$(echo ${HOST_NAME} | grep bg)
if [ -z "${BG_VALUE}" ]; then
  BG_VALUE=""
else
  BG_VALUE="bg"
  echo "BG_VALUE = ${BG_VALUE}"
fi
#PORT=${SERVER_PORT}+1
PORT=$(expr ${SERVER_PORT} + 1)
STAGE_DIR=$AppRoot/stage
ARCHIVE_DIR=$AppRoot/archive
CONFIGMAP_DIR=$AppRoot/latest
SCRIPTS_DIR=$AppRoot/upscripts
EXTRACT_DIR=$AppRoot/extract
NOW=$(date +"%m%d%Y%H%M")
SENV=$(echo ${ENV} | tr -dc '0-9')
RELEASE=${VERSION##*/}
export HOST_NAME=${HOST_NAME}
OS=$(uname -s |  tr '[:upper:]' '[:lower:]')
if [[ "${OS}" == "sunos" ]]
then
  HOST_IP=`/sbin/ifconfig -a | grep 'inet'  | grep -v '127.0.0.1'|awk '{print $2}' | awk 'NR==1{print $1}'`
elif [[ "${OS}" == "linux" ]]
then
  HOST_IP=`hostname -i`
fi

JavaOpts=$(echo ${JavaOpts} | sed -e "s|{{hostname}}|${HOSTNAME}|g" | sed -e "s|{{host_CONFIG}}|${HOST_NAME}|g" | sed -e "s|{{SENV}}|${SENV}|g" | sed -e "s|{{BRANCH}}|${VERSION}|g" | sed -e "s|{{BGVALUE}}|${BG_VALUE}|g" | sed -e "s|{{hostIp}}|${HOST_IP}|g" | sed -e "s|{{registry}}|${REGISTRY}|g"  | sed -e "s|{{RELEASE}}|${RELEASE}|g")
# Source env
. ${SCRIPTS_DIR}/common.env ${DEPLOY_ENV} ${AppEnv} ${JavaOpts}

USE_GREP="${USE_GREP:=grep}"
USE_SED="${USE_SED:=sed}"
TIMESTAMP_VALUE="${TIMESTAMP_VALUE:=-}"
if [ -z "${DEPLOY_DIR}" ]; then
  DEPLOY_DIR=$AppRoot/latest
else
  DEPLOY_DIR=$DEPLOY_DIR
fi

if [ -z "${SVC_NAME}" ]; then
  APP_NAME=$APP_NAME
else
  APP_NAME=${SVC_NAME}
fi

file_ext="${EXTENSION:=jar}"
curr_work_dir="${CURR_WORKING_DIR}"
INSTANCE_NAME="${INSTANCE_NAME:=$APP_NAME}"
BASE_DIR=`dirname ${DEPLOY_DIR}`
APP_DIR=`basename ${DEPLOY_DIR}`

echo "================ JAR Variable Info ===================="
echo "LOGS_DIR: $LOGS_DIR"
echo "SCRIPTS_DIR: $SCRIPTS_DIR"
echo "DEPLOY_DIR: $DEPLOY_DIR"
echo "ARCHIVE_DIR: $ARCHIVE_DIR"
echo "SERVER_PORT: $SERVER_PORT"
echo "INSTANCE_NAME: $INSTANCE_NAME"  
echo "INSTANCE_COUNT: $INSTANCE_COUNT" 
echo "INSTANCE_START_INDEX: $INSTANCE_START_INDEX" 
echo "JavaOptsENV: $JavaOptsENV"
echo "ARTIFACT_TYPE: $file_ext"
echo "DEPLOY_ENV: $DEPLOY_ENV"
echo "Config files: ${cm_src_dest}"
echo "TIMESTAMP_VALUE: $TIMESTAMP_VALUE"
echo "========================================================"

[ -d $SCRIPTS_DIR ] || mkdir -p $SCRIPTS_DIR;
[ -d $CONFIGMAP_DIR ] || mkdir -p $CONFIGMAP_DIR;
[ -d $STAGE_DIR/$APP_NAME ] || mkdir -p $STAGE_DIR/$APP_NAME;
[ -d $ARCHIVE_DIR/$APP_NAME ] || mkdir -p $ARCHIVE_DIR/$APP_NAME;
[ -d $DEPLOY_DIR/$APP_NAME ] || mkdir -p $DEPLOY_DIR/$APP_NAME;
[ -d $EXTRACT_DIR/$APP_NAME ] || mkdir -p $EXTRACT_DIR/$APP_NAME;

cd $STAGE_DIR/$APP_NAME;
rm -rf $STAGE_DIR/$APP_NAME/${APP_NAME}*.$file_ext

if [ ${VSAD} == 'evbv' ];
  then
    wget -O ${APP_NAME}.zip ${ARTIFACT_URL} -q
else
 wget ${ARTIFACT_URL} -q
fi
if [[ $? -ne 0 ]]; then
    echo "Artifact: ${ARTIFACT_URL} download failed";
    exit 1; 
fi
echo "Artifact: ${ARTIFACT_URL} downloaded";
echo "STAGED FILE:   $(ls -l $STAGE_DIR/$APP_NAME/${APP_NAME}*.$file_ext)"

#Stopping instance
if [ ${RESTART} == 'yes' ]; then
  if [ -e $STAGE_DIR/$APP_NAME/${APP_NAME}*.$file_ext ]
  then
    echo "Stopping ${INSTANCE_NAME}......"
    $SCRIPTS_DIR/stop.sh $INSTANCE_NAME $SCRIPTS_DIR $SERVER_PORT $INSTANCE_COUNT $ENV $MEM $VSAD $AppRoot $JavaOpts $AppEnv $HOST_NAME $VERSION 'no'
  else
     exit 1;
  fi
fi

echo "Removing the old back up files"
echo "$(ls -ltr $ARCHIVE_DIR/$APP_NAME/)"
rm -rf $ARCHIVE_DIR/$APP_NAME/*

#Create backup of existing files
if [[ -n "${BACKUP}" && "${BACKUP}" == "no" ]];
then
  echo "No backup created for ${APP_NAME}......"
elif [[ $DEPLOY_FILE_NAME && -n "${DEPLOY_FILE_NAME}" ]]; 
then
  if [ -e $DEPLOY_DIR/${DEPLOY_FILE_NAME}.jar ];
  then
    mv $DEPLOY_DIR/${DEPLOY_FILE_NAME}.jar $ARCHIVE_DIR/$APP_NAME/;
    echo "Backup file: $(ls -l  $ARCHIVE_DIR/$APP_NAME/${DEPLOY_FILE_NAME}.jar)";
  else
    echo "Jar ${DEPLOY_FILE_NAME} not found for backup: $( ls -l $DEPLOY_DIR/${DEPLOY_FILE_NAME}.jar)";
  fi
elif [ -e $DEPLOY_DIR/$APP_NAME/${APP_NAME}*.jar ]
then
  mv $DEPLOY_DIR/$APP_NAME/${APP_NAME}*.jar $ARCHIVE_DIR/$APP_NAME/;
  echo "Backup file: $( ls -l $ARCHIVE_DIR/$APP_NAME/${APP_NAME}*.jar)";
elif [ -e $DEPLOY_DIR/$APP_NAME/${APP_NAME}*.war ]
then
  mv $DEPLOY_DIR/$APP_NAME/${APP_NAME}*.war $ARCHIVE_DIR/$APP_NAME/;
  echo "Backup file: $( ls -l $ARCHIVE_DIR/$APP_NAME/${APP_NAME}*.war)";
elif [ ${file_ext} == 'zip' ];
then
  [ -d $ARCHIVE_DIR/$APP_NAME/$APP_DIR ] || mkdir -p $ARCHIVE_DIR/$APP_NAME/$APP_DIR;
  if [[ $DEPLOY_FOLDER_NAME && -n "${DEPLOY_FOLDER_NAME}" && "${DEPLOY_FOLDER_NAME}" != "${APP_DIR}" ]]; then
    mv $DEPLOY_DIR/$DEPLOY_FOLDER_NAME/* $ARCHIVE_DIR/$APP_NAME/$APP_DIR;
  else
    mv $DEPLOY_DIR/* $ARCHIVE_DIR/$APP_NAME/$APP_DIR;
  fi
  zip -qr -D $ARCHIVE_DIR/$APP_NAME/${APP_NAME}_$NOW.zip $ARCHIVE_DIR/$APP_NAME/$APP_DIR;
  rm -rf $ARCHIVE_DIR/$APP_NAME/$APP_DIR
  echo "Backup file: $(ls -l  $ARCHIVE_DIR/$APP_NAME/${APP_NAME}_$NOW.zip)";
else
  echo "File type not supported for backup";
fi

#Deploy artifact
if [ ${file_ext} == 'zip' ]; 
then
  cd ${EXTRACT_DIR}/${APP_NAME}
  rm -rf ${EXTRACT_DIR}/${APP_NAME}/*
  cp ${STAGE_DIR}/$APP_NAME/${APP_NAME}*.zip .
  export  UNZIP_DISABLE_ZIPBOMB_DETECTION=TRUE
  unzip -qo ${APP_NAME}*.zip
  rm -rf ${APP_NAME}*.zip
  if [ ${VSAD} == 'evbv' ];
  then
    mv ${APP_NAME}* ${APP_NAME}
  fi
  echo "Copying contents from  ${EXTRACT_DIR}/${APP_NAME}......"
  
  if [[ $DEPLOY_FOLDER_NAME && -n "${DEPLOY_FOLDER_NAME}" ]]; then
    [ -d $DEPLOY_DIR/$DEPLOY_FOLDER_NAME ] || mkdir -p $DEPLOY_DIR/$DEPLOY_FOLDER_NAME;
    if [[ -d "${EXTRACT_DIR}/${APP_NAME}/${APP_DIR}" && -n "$(ls ${EXTRACT_DIR}/${APP_NAME}/${APP_DIR})" && "${DEPLOY_FOLDER_NAME}" != "${APP_DIR}" ]]; then
      cp -r ${EXTRACT_DIR}/${APP_NAME}/${APP_DIR}/* ${DEPLOY_DIR}/${DEPLOY_FOLDER_NAME}
    elif [[ -d "${EXTRACT_DIR}/${APP_NAME}/${APP_DIR}" && -n "$(ls ${EXTRACT_DIR}/${APP_NAME}/${APP_DIR})" && "${DEPLOY_FOLDER_NAME}" == "${APP_DIR}" ]]; then
      cp -r ${EXTRACT_DIR}/${APP_NAME}/${APP_DIR}/* ${DEPLOY_DIR}
    elif [[ -n "$(ls ${EXTRACT_DIR}/${APP_NAME}/${INSTANCE_NAME}*)" && "${DEPLOY_FOLDER_NAME}" == "${APP_DIR}" ]]; then
      cp -r ${EXTRACT_DIR}/${APP_NAME}/${INSTANCE_NAME}*/* ${DEPLOY_DIR}
    elif [[ -n "$(ls ${EXTRACT_DIR}/${APP_NAME}/${INSTANCE_NAME}*)" && "${DEPLOY_FOLDER_NAME}" != "${APP_DIR}" ]]; then
      cp -r ${EXTRACT_DIR}/${APP_NAME}/${INSTANCE_NAME}*/* ${DEPLOY_DIR}/${DEPLOY_FOLDER_NAME}
    elif [[ -d "${EXTRACT_DIR}/${APP_NAME}" && -n "$(ls ${EXTRACT_DIR}/${APP_NAME})" ]]; then
      cp -r ${EXTRACT_DIR}/${APP_NAME}/* ${DEPLOY_DIR}/${DEPLOY_FOLDER_NAME}
    else
      echo "No content extracted from zip: $(ls -l  ${EXTRACT_DIR}/${APP_NAME})"
      exit 1
    fi
  elif [[ -d "${EXTRACT_DIR}/${APP_NAME}/${APP_DIR}" && -n "$(ls ${EXTRACT_DIR}/${APP_NAME}/${APP_DIR})" ]]; then
    cp -r ${EXTRACT_DIR}/${APP_NAME}/${APP_DIR}/* ${DEPLOY_DIR}
  else
    cp -r ${EXTRACT_DIR}/${APP_NAME}/* ${DEPLOY_DIR}
  fi
  rm -rf ${EXTRACT_DIR}/${APP_NAME}/*
elif [ ${file_ext} == 'tar' ]; 
then
  cd ${DEPLOY_DIR}/${APP_NAME}
  cp ${STAGE_DIR}/$APP_NAME/${APP_NAME}*.tar .
  tar xf ${APP_NAME}*.tar
  rm -rf ${APP_NAME}*.tar
  if [[ $DEPLOY_FILE_NAME && -n "${DEPLOY_FILE_NAME}" ]]; then 
    if [ -e $DEPLOY_DIR/$APP_NAME/${APP_NAME}*.jar ];
    then
      mv $DEPLOY_DIR/$APP_NAME/${APP_NAME}*.jar $DEPLOY_DIR/${DEPLOY_FILE_NAME}.jar;
    fi
  fi
elif [ ${file_ext} == 'war' ]; 
then
  cd ${DEPLOY_DIR}/${APP_NAME}
  cp ${STAGE_DIR}/$APP_NAME/${APP_NAME}*.war .
elif [[ $DEPLOY_FILE_NAME && -n "${DEPLOY_FILE_NAME}" ]]; 
then
  if [ -e $STAGE_DIR/$APP_NAME/${APP_NAME}*.jar ]; 
  then
    mv $STAGE_DIR/$APP_NAME/${APP_NAME}*.jar $DEPLOY_DIR/${DEPLOY_FILE_NAME}.jar;
    cd $DEPLOY_DIR;
  fi
else
  #removing existing file
  rm -rf $DEPLOY_DIR/$APP_NAME/${APP_NAME}*.$file_ext
  mv $STAGE_DIR/$APP_NAME/${APP_NAME}*.$file_ext $DEPLOY_DIR/$APP_NAME;
fi

# Exclude files
if [[ "$EXCLUDE_FILES" && -n "${EXCLUDE_FILES}" ]]; 
then
  for file_item in ${EXCLUDE_FILES//,/ } ; do
    if [[ -d "${file_item%/*}" && -n "$(ls ${file_item%/*})" ]]; then
      echo "Removing file: ${file_item}"
      rm -rf ${file_item}
    else
      echo "File pattern : ${file_item} : doesn't exist"
    fi
  done
fi

# Deployment validation
if [[ "$DEPLOY_FILE_NAME" && -n "${DEPLOY_FILE_NAME}" ]]; 
then
  if [ -e ${DEPLOY_DIR}/${DEPLOY_FILE_NAME}.jar  ];
  then
    echo "Deployment files found for: ${APP_NAME}"
    echo "Deployment file: $(ls -l ${DEPLOY_DIR}/${DEPLOY_FILE_NAME}.jar )"
  else
    echo "Deployment files not found for: ${APP_NAME}"
    exit 1
  fi
elif [ -e $DEPLOY_DIR/$APP_NAME/${APP_NAME}*.jar ];
then
  echo "Deployment files found for: ${APP_NAME}"
  echo "Deployment file: $(ls -l $DEPLOY_DIR/$APP_NAME/${APP_NAME}*.jar )"
elif [ -e $DEPLOY_DIR/$APP_NAME/${APP_NAME}*.war ];
then
  echo "Deployment files found for: ${APP_NAME}"
  echo "Deployment file: $(ls -l $DEPLOY_DIR/$APP_NAME/${APP_NAME}*.war )"
elif [ ${file_ext} == 'zip' ];
then
  if [[ -d "${DEPLOY_DIR}" && -n "$(ls $DEPLOY_DIR)" ]]; 
  then
    echo "Deployment files found for: ${APP_NAME}"
    echo "Deployment file: $(ls -l $DEPLOY_DIR )"
  else
    echo "Deployment files not found for: ${APP_NAME}"
    exit 1
  fi
else
  echo "Deployment files not found for: ${APP_NAME}"
  exit 1
fi

#Config deployment
if [ "$cm_src_dest" != "-" ]; then
  echo "Config files found for: ${APP_NAME}"
  for cm_item in ${cm_src_dest//,/ } ; do
    tmpKey=`echo ${cm_item} | cut -d ":" -f 1`
    tmpVal=`echo ${cm_item} | cut -d ":" -f 2`
    echo "SRC: ${tmpKey}, DEST: ${tmpVal}"
    if [ -d $tmpKey ]; then
      cp -r ${tmpKey} ${tmpVal}
    elif [ -e $tmpKey ]; then
      cp ${tmpKey} ${tmpVal}
      echo "${USE_SED} -i \"s|{{APP_NAME}}|$APP_NAME|g;s|{{SERVER_PORT}}|$PORT|g;s|{{ENV}}|$ENV|g;s|TIMESTAMP_VALUE|$TIMESTAMP_VALUE|g;s|{{LOGS_DIR}}|$LOGS_DIR|g\" ${tmpVal}"
      ${USE_SED} -i "s|{{APP_NAME}}|$APP_NAME|g;s|{{SERVER_PORT}}|$PORT|g;s|{{ENV}}|$ENV|g;s|TIMESTAMP_VALUE|$TIMESTAMP_VALUE|g;s|{{LOGS_DIR}}|$LOGS_DIR|g" ${tmpVal}
    else
      echo "SRC config file doesn't exist"
    fi
    if [[ $? -ne 0 ]]; then
      echo "Config file copy failed";
    fi
  done
fi

#Starting instance
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

echo "Deployment completed for: ${APP_NAME}"

S3_UPLOAD_CMD="${S3_UPLOAD_CMD:=-}"
if [ "${S3_UPLOAD_CMD}" != '-' ]; then
  $S3_UPLOAD_CMD
fi

tmp_Artifact_url=`echo ${ARTIFACT_URL} | awk -F"artifactory/" '{print $2}'`
ARTIFACT_INFO_FILE_NAME="${ARTIFACT_INFO_FILE_NAME:=-}"
ARTIFACT_INFO_APP_NAME="${ARTIFACT_INFO_APP_NAME:=$APP_NAME}"
ARTIFACT_INFO_DEPLOY_DIR="${ARTIFACT_INFO_DEPLOY_DIR:=$DEPLOY_DIR}"
if [ ${ARTIFACT_INFO_FILE_NAME} != '-' ]; then
  grep -q "^$ARTIFACT_INFO_APP_NAME:" $ARTIFACT_INFO_DEPLOY_DIR/$ARTIFACT_INFO_FILE_NAME && ${USE_SED} -i "s/^$ARTIFACT_INFO_APP_NAME:.*/$ARTIFACT_INFO_APP_NAME:${tmp_Artifact_url}/" $ARTIFACT_INFO_DEPLOY_DIR/$ARTIFACT_INFO_FILE_NAME || echo "$ARTIFACT_INFO_APP_NAME:${tmp_Artifact_url}" >> $ARTIFACT_INFO_DEPLOY_DIR/$ARTIFACT_INFO_FILE_NAME 
  echo "Artifact info file updated => $ARTIFACT_INFO_APP_NAME:${tmp_Artifact_url}"
fi

