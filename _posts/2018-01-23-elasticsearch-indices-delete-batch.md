---
layout: post
title:  "Elasticsearch 특정 갯수의 indices/alias 남기고 삭제하는 배치"
date:   2018-01-23
categories: elasticsearch
tags: elasticsearch batch shell
excerpt: 엘라스틱 서치를 사용하다 보면 정기적으로 인덱스를 삭제하거나 alias를 정기적으로 변경해야 할 경우가 발생합니다.아래의 shell script 는 그런 경우에 사용하기 위해서 작성한 것입니다. 
mathjax: true
---

Elasticsearch > batch
===================
엘라스틱 서치를 사용하다 보면 정기적으로 인덱스를 삭제하거나 alias를 정기적으로 변경해야 할 경우가 발생합니다.
아래의 shell script 는 그런 경우에 사용하기 위해서 작성한 것입니다. 


```
#!/bin/bash
################################################
# Execution environment define
################################################
# Its depend on Env.: Dev, Stg, Qa, Prod
TARGET_SERVER="localhost:9200"
# DevOps Group will be define.
HOME_DIR="/home/logstash"
# For logging
RUN_DATE=`date +%Y-%m-%d`
# It will rotate per day
LOG_FILE="${HOME_DIR}/logs/apache-index-controller.log.$RUN_DATE"
# It temporary file for processing, Its will reset by every running.
APACHE_GROUP_FILE="/${HOME_DIR}/APACHE_GROUP_FILE.txt"
TARGET_INDEX_FILE="/${HOME_DIR}/TARGET_INDEX_FILE.txt"
# It defines, how many the Indices remain per each Popularity pattern
MAX_INDEX_NUMBER=46
# It defines, Offset for "close" and "delete" timing
MAX_INDEX_NUMBER_CLOSE_OFFSET=1

################################################
# Start
################################################
touch $LOG_FILE
echo "#######################################">>$LOG_FILE
echo `date '+%Y-%m-%d %H:%M:%S'`":Start">>$LOG_FILE
echo "#######################################">>$LOG_FILE

################################################
# Get all Indices for make apache group
################################################
#remove old data
`rm -f $APACHE_GROUP_FILE`
#Create blank file
`touch $APACHE_GROUP_FILE`
#Get Target Index list : example out : apache-tag apache-access apache-item
RUN_COMMAND="curl -XGET \"http://${TARGET_SERVER}/_cat/indices/apache*\" 2>/dev/null | /usr/bin/awk '{ print \$3 }' | sort -nr | awk -F \"-\" '{printf \$1\"-\"\$2 \"\n\"}' | uniq"
echo `eval ${RUN_COMMAND}` >>$APACHE_GROUP_FILE

#convert file to Array
TEMP=`cat ${APACHE_GROUP_FILE}`
INDEX_PATTERN_ARRAY=(${TEMP})
echo "Target PATTERNS:" >> $LOG_FILE
echo ${INDEX_PATTERN_ARRAY[0]} >> $LOG_FILE
echo ${INDEX_PATTERN_ARRAY[1]} >> $LOG_FILE
echo ${INDEX_PATTERN_ARRAY[2]} >> $LOG_FILE
echo "----------------------------">> $LOG_FILE

################################################
# Looping for processing by each Popularity group
################################################
for idx_pattern in "${INDEX_PATTERN_ARRAY[@]}"
do
  #remove old data
  `rm -f $TARGET_INDEX_FILE`
  #Create blank file
  `touch $TARGET_INDEX_FILE`
  #get all indices with fixed prefix(idx_pattern) and sorting
  RUN_COMMAND="curl -XGET \"http://${TARGET_SERVER}/_cat/indices/${idx_pattern}*\" 2>/dev/null | /usr/bin/awk '{ print \$3 }' | sort -nr"
  echo `eval ${RUN_COMMAND}` >> $TARGET_INDEX_FILE

  TEMP=`cat ${TARGET_INDEX_FILE}`
  TMP_ARRARY=($TEMP)
  X=1
  ################################################
  # For add to aliases
  ################################################
  for target_index in "${TMP_ARRARY[@]}"
  do
    #TODO Add Alias
    RUN_COMMAND="curl -XPOST \"http://${TARGET_SERVER}/_aliases\" -H 'Content-Type: application/json' -d' { \"actions\" : [{ \"add\" : { \"index\" : \"${target_index}\", \"alias\" : \"${idx_pattern}\" } }]}'"
    status=`eval ${RUN_COMMAND}`
    echo "[Add to aliases] :${idx_pattern} Target Index: ${target_index} Result:${status}" >> $LOG_FILE
    #if exceed MAX_INDEX_NUMBER, its will break
    if [ $MAX_INDEX_NUMBER -eq $X ]
    then
       echo "break due to reach maxt index number:$X" >> $LOG_FILE
       break
    fi
    X=$(( $X + 1 ))
  done

  ################################################
  # For remove from aliases
  ################################################
  X=1
  for target_index in "${TMP_ARRARY[@]}"
  do
    if [ $MAX_INDEX_NUMBER -lt $X ]
    then
      RUN_COMMAND="curl -XPOST \"http://${TARGET_SERVER}/_aliases\" -H 'Content-Type: application/json' -d' { \"actions\" : [{ \"remove\" : { \"index\" : \"${target_index}\", \"alias\" : \"${idx_pattern}\" } }]}'"
      status=`eval ${RUN_COMMAND}`
      echo "[Remove from aliases] :${idx_pattern} Target Index: ${target_index} Result:${status}" >> $LOG_FILE
    fi
    X=$(( $X + 1 ))
  done

  ################################################
  # For close and delete index over (MAX_INDEX_NUMBER + MAX_INDEX_NUMBER_CLOSE_OFFSET) number.
  ################################################
  # for close index : MAX_INDEX_NUMBER + 1
  X=1
  for target_index in "${TMP_ARRARY[@]}"
  do
    if [ $(($MAX_INDEX_NUMBER + $MAX_INDEX_NUMBER_CLOSE_OFFSET)) -le $X ]
    then
      RUN_COMMAND="curl -XPOST \"http://${TARGET_SERVER}/${target_index}/_close\""
      status=`eval ${RUN_COMMAND}`
      echo "[Close and Delete index] :${target_index} Status:${status}" >> $LOG_FILE
      curl -XDELETE "http://${TARGET_SERVER}/${target_index}"
    fi
    X=$(( $X + 1 ))
  done
done

################################################
# Done.
################################################
echo "--------------------------------" >>$LOG_FILE
echo `date '+%Y-%m-%d %H:%M:%S'`":Done" >>$LOG_FILE
echo "--------------------------------" >>$LOG_FILE
echo ""
exit 0
```