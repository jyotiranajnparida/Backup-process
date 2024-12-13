#set -e #stops execution if a variable is not set
#set -u #stop execution if something goes wrong
set -x
usage() {
echo "usage: $(basename $0) [option]"
echo "option=full: do a full backup using innobackupex"
#echo "option=incremental: do a incremental backup"
echo "option=diff: do a differential backup"
echo "option=help: show this help"
}
full_backup() {
### CHECK COUNT OF FULL BACKUPS IN CURRENT WEEK
################
FULL_BKP_STS=`$MYSQL "select count(1) from dbbkpstatus where
ip_address='$IP_ADDR' and WEEK=$wk and bkp_type='FULL';"`
if [ $FULL_BKP_STS -eq 0 ];
then
NUMBER=1;
else
PFN=`$MYSQL "select bkp_number from dbbkpstatus where
ip_address='$IP_ADDR' and bkp_type='FULL' and WEEK=$wk order by
bkp_number desc limit 1"`
NUMBER=$(($PFN + 1))
fi
##### Check log directory existence ##########
if [ ! -d $LOG_DIR ]
then
mkdir -p $LOG_DIR
if [ $? -ne 0 ]
then
echo "Full Backup failure.Cannot create log directory at
$LOG_DIR." | mailx -s "MYSQL $1 Backup Failure for component -
$component in $IP_ADDR" -a "$LOG_DIR/${PRFX}.log" $recipients
exit 1
fi
date >> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
else
echo "Log directory check successful." >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
fi
date >> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
##### Check S3 connection #####
echo "S3 dryrun write validation running.." >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
touch $LOG_DIR/write.txt
aws s3 cp $LOG_DIR/write.txt "s3://$bucket_name" --dryrun >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1 ## directly add S3_BKP_BKT
to this --pending
date >> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
if [ $? -ne 0 ]
then
echo "Check !! dryrun write failed in S3 bucket- $S3_BKP_BKT" >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
rm $LOG_DIR/write.txt
echo "Full Backup failure.Check !! dryrun write failed in S3 bucket-
$S3_BKP_BKT"| mailx -s "MYSQL $1 Backup Failure for component -
$component in $IP_ADDR" -a "$LOG_DIR/${PRFX}_full${NUMBER}.log"
$recipients
exit 1
else
echo "dryrun write on S3 successful." >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
rm $LOG_DIR/write.txt
fi
date >> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
##### CHECK LSN DIRECTORY EXISTENCE ###########
echo "Checking if lsn directory exists $lsndir ..." >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
if [ ! -d $lsndir ]
then
echo "ERROR: the folder $lsndir does not exist" >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
date >> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
echo "Trying to create $lsndir" >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
mkdir -p $lsndir 2>> $LOG_DIR/${PRFX}_full${NUMBER}.log
if [ $? -ne 0 ]
then
echo "ERROR: Cannot create folder $lsndir" >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
echo "Full Backup failure.ERROR: Cannot create folder
$lsndir" | mailx -s "MYSQL $1 Backup Failure for component - $component in
$IP_ADDR" -a "$LOG_DIR/${PRFX}_full${NUMBER}.log" $recipients
exit 1
else
echo "$lsndir created successfully" >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
fi
date >> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
else
echo "Directory check successful." >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
fi
date >> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
##### INITIATING FULL BACKUP #######################
$MYSQL "update dbbkpstatus set
bkp_type='FULL',bkp_number=$NUMBER,local_bkp='$S3_BKP_PATH/${PRF
X}_FULL${NUMBER}' where ip_address='$IP_ADDR' and
bkp_name='$BKP_NAME'" >> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
echo "Initiating full backup Number ${NUMBER}..." >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
xtrabackup --backup --stream=xbstream
--target-dir=$lsndir/${PRFX}_FULL${NUMBER}
--extra-lsndir=$lsndir/${PRFX}_FULL${NUMBER} $ARGS 2>
$LOG_DIR/xtrabackup-${TS}_FULL${NUMBER}.log |xbcloud put
$xbcloud_backup/${PRFX}_FULL${NUMBER} --storage=s3
--s3-bucket=$bucket_name --s3-region='ap-south-1'
--s3-access-key=AKIAVOMR2TMUNZ3B56WL
--s3-secret-key=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
--parallel=1 2> $LOG_DIR/xbcloud-${TS}-S3UPLOAD${NUMBER}-stderr.log
stat=`echo ${PIPESTATUS[0]} ${PIPESTATUS[1]}` ## capture status of
xtrabackup|xbcloud
xtrabackup_status=`echo $stat|awk '{print $1}'`
xbcloud_status=`echo $stat|awk '{print $1}'`
date >> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
if [[ $xtrabackup_status -ne 0 || $xbcloud_status -ne 0 ]];
then
error_log=`tail -n 2
$LOG_DIR/xtrabackup-${TS}_FULL${NUMBER}.log
$LOG_DIR/xbcloud-${TS}-S3UPLOAD${NUMBER}-stderr.log` ## check the
size of "remarks" column in table "dbbkpstatus"
mysql -u'open_user' -pF0rB@kPUse -h'10.100.64.53' inventory
-Bse "update dbbkpstatus set status='FAILED' , end_date=now(),
remarks='$error_log' where ip_address='$IP_ADDR' and
bkp_name='$BKP_NAME'" >> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
date >> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
if [ $xtrabackup_status -ne 0 ]
then
echo "xtrabackup failed." >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
fi
if [ $xbcloud_status -ne 0 ]
then
echo "xbcloud S3 upload failed." >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
fi
echo "Full Backup Failure running xtrabackup command."| mailx -s
"MYSQL $1 Backup Failure for component - $component in $IP_ADDR" -a
"$LOG_DIR/${PRFX}_full${NUMBER}.log" -a
"$LOG_DIR/xtrabackup-${TS}_FULL${NUMBER}.log" -a
"$LOG_DIR/xbcloud-${TS}-S3UPLOAD${NUMBER}-stderr.log" $recipients
cat $LOG_DIR/xtrabackup-${TS}_FULL${NUMBER}.log >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
cat $LOG_DIR/xbcloud-${TS}-S3UPLOAD${NUMBER}-stderr.log
>> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
echo $error_log >> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
rm $LOG_DIR/xtrabackup-${TS}_FULL${NUMBER}.log
rm $LOG_DIR/xbcloud-${TS}-S3UPLOAD${NUMBER}-stderr.log
exit; exit;
fi
if [[ ! -z `tail -1 $LOG_DIR/xtrabackup-${TS}_FULL${NUMBER}.log | grep
'completed OK!'` && ! -z `tail -1
$LOG_DIR/xbcloud-${TS}-S3UPLOAD${NUMBER}-stderr.log | grep 'Upload
completed'` ]];
then
cat $LOG_DIR/xtrabackup-${TS}_FULL${NUMBER}.log >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
cat $LOG_DIR/xbcloud-${TS}-S3UPLOAD${NUMBER}-stderr.log
>> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
rm $LOG_DIR/xtrabackup-${TS}_FULL${NUMBER}.log
rm $LOG_DIR/xbcloud-${TS}-S3UPLOAD${NUMBER}-stderr.log
echo "Percona Full Backup & S3 Upload Completed Successfully"
>> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
date >> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
TO_LSN=`cat
$lsndir/${PRFX}_FULL${NUMBER}/xtrabackup_checkpoints | grep to_lsn |
awk -F" = " '{print$2}'`
FROM_LSN=`cat
$lsndir/${PRFX}_FULL${NUMBER}/xtrabackup_checkpoints | grep from_lsn |
awk -F" = " '{print$2}'`
$MYSQL "update dbbkpstatus set status='COMPLETED',
bkp_type='FULL', bkp_number=${NUMBER}, from_lsn=$FROM_LSN,
to_lsn='$TO_LSN', end_date=now(),
s3_bkp_path='$S3_BKP_BKT/$YEAR/WEEK$WEEK/$S3_BKP_DIR/$IP_AD
DR/${PRFX}_FULL${NUMBER}',s3_bkp_dir='$S3_BKP_DIR' where
ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
date >> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
transac=`$MYSQL "select is_transactional from dbinfra where
ip_address='$IP_ADDR'"`
if [[ $transac == 'Yes' ]];
then
MNTH_FULL_BKP_CNT=`$MYSQL "select count(*) from
dbbkpstatus where ip_address='$IP_ADDR' and bkp_type='FULL' and
MONTH=$MONT$YEAR and status='COMPLETED'"`
if [[ $MNTH_FULL_BKP_CNT == 1 ]]
then
perstbkt=`$MYSQL "select s3_persistant_bkp_bucket
from dbinfra where ip_address='$IP_ADDR'"` ##s3://sd-db-persist-backup &
s3://sd-db-persistent-backup
cmpfullbkppath=`$MYSQL "select s3_bkp_path from
dbbkpstatus where ip_address='$IP_ADDR' and bkp_type='FULL' and
MONTH=$MONT$YEAR and status='COMPLETED'"`
##s3://mysql-trans-dbs/2023/WEEK49/NULL/10.100.56.101/Xtracmprsbkp_ap
igateway-mysqlapi-8001_2023-12-05-13-15-49_FULL21
cmpfullbkpname=`echo $cmpfullbkppath | rev | cut -f1 -d'/'
| rev`
##Xtracmprsbkp_apigateway-mysqlapi-8001_2023-12-05-13-15-49_FULL21
$MYSQL "insert into
dbbkpstatus_mnthly(ip_address,hostname,status,MONTH,s3_bkp_path,start_
date)
values('$IP_ADDR','$HST_NAME','STARTED','$MONT$YEAR','${perstbkt}/$Y
EAR/MONTH${MONT}/$S3_BKP_DIR/$cmpfullbkpname',now())"
aws s3 sync $cmpfullbkppath/
${perstbkt}/$YEAR/MONTH${MONT}/$S3_BKP_DIR/$cmpfullbkpname/ >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
if [[ $? -eq 0 ]];
then
$MYSQL "update dbbkpstatus_mnthly set
status='COMPLETED',end_date=now() where ip_address='$IP_ADDR' and
MONTH=$MONT$YEAR;"
else
$MYSQL "update dbbkpstatus_mnthly set
status='FAILED',end_date=now() where ip_address='$IP_ADDR' and
MONTH=$MONT$YEAR;"
fi
else
echo "No more Persistant backup uploads required this
month." >> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
fi
fi
else
date >> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
error_log=`tail -n 2
$LOG_DIR/xtrabackup-${TS}_FULL${NUMBER}.log
$LOG_DIR/xbcloud-${TS}-S3UPLOAD${NUMBER}-stderr.log`
cat $LOG_DIR/xtrabackup-${TS}_FULL${NUMBER}.log >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
cat $LOG_DIR/xbcloud-${TS}-S3UPLOAD${NUMBER}-stderr.log
>> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
echo
"#############################################################
#########" >> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
echo "Percona Full Backup Number ${NUMBER} Failed" >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
echo "#### ERROR HIGHLIGHTS ####" >>
$LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
echo $error_log >> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
rm $LOG_DIR/xtrabackup-${TS}_FULL${NUMBER}.log
rm $LOG_DIR/xbcloud-${TS}-S3UPLOAD${NUMBER}-stderr.log
echo "FULL Backup Failure in xtrabackup command.Review
attached log for details"| mailx -s "MYSQL $1 Backup Failure for component -
$component in $IP_ADDR" -a "$LOG_DIR/${PRFX}_full${NUMBER}.log"
$recipients
echo
"#############################################################
#########" >> $LOG_DIR/${PRFX}_full${NUMBER}.log 2>&1
mysql -u'open_user' -pF0rB@kPUse -h'10.100.64.53' inventory
-Bse "update dbbkpstatus set status='FAILED', bkp_type='FULL' and
bkp_number=${NUMBER}, end_date=now() where ip_address='$IP_ADDR'
and bkp_name='$BKP_NAME'" >> $LOG_DIR/${PRFX}_full${NUMBER}.log
2>&1
fi
#rm -rf $BACKUP_DIR/xtra.sock
}
differential_backup(){
FULL_BKP_STS=`$MYSQL "select count(1) from dbbkpstatus where
ip_address='$IP_ADDR' and WEEK=$wk and bkp_type='FULL' and
status='COMPLETED'"`
if [[ $FULL_BKP_STS == 0 ]];
then
echo "ERROR: No Succesfull Full backup has been done before
this week. aborting" >> $LOG_DIR/${PRFX}.log 2>&1
$MYSQL "update dbbkpstatus set status='FAILED',
bkp_type='DIFFERENTIAL', start_date=now(), end_date=now(), remarks='No
Succesfull Full backup has been done before this week.' where
ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >>
$LOG_DIR/${PRFX}.log 2>&1
echo "ERROR: No Succesfull Full backup has been done before
this week. aborting DIFF backup"| mailx -s "MYSQL $1 Backup Failure for
component - $component in $IP_ADDR" -a "$LOG_DIR/${PRFX}.log"
$recipients
exit;
else
LFN=`$MYSQL "select bkp_number from dbbkpstatus where
ip_address='$IP_ADDR' and bkp_type='FULL' and WEEK=$wk and
status='COMPLETED' order by bkp_number desc limit 1"`
LFNAME=`$MYSQL "select bkp_name from dbbkpstatus where
ip_address='$IP_ADDR' and bkp_type='FULL' and WEEK=$wk and
status='COMPLETED' order by bkp_number desc limit 1"`
fi
## FRAMING CURRENT DIFFERENTIAL NUMBER ###
DIFF_BKP_STS=`$MYSQL "select count(1) from dbbkpstatus where
ip_address='$IP_ADDR' and WEEK=$wk and ref_full_bkp='FULL$LFN' and
bkp_type='DIFFERENTIAL'"`
if [[ $DIFF_BKP_STS == 0 ]];
then
NUMBER=1;
else
LDN=`$MYSQL "select bkp_number from dbbkpstatus where
ip_address='$IP_ADDR' and WEEK=$wk and ref_full_bkp='FULL$LFN' and
bkp_type='DIFFERENTIAL' order by bkp_number desc limit 1"`
NUMBER=$(($LDN + 1))
fi
##### Check log directory existence ##########
if [ ! -d $LOG_DIR ]
then
mkdir -p $LOG_DIR
if [ $? -ne 0 ]
then
echo "DIFF Backup failure.Cannot create log directory at
$LOG_DIR." | mailx -s "MYSQL $1 Backup Failure for component -
$component in $IP_ADDR" -a "$LOG_DIR/${PRFX}.log" $recipients
exit 1
fi
date >> $LOG_DIR/${PRFX}.log 2>&1
else
echo "Log directory check successful." >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
fi
date >> $LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
##### Check S3 connection #####
echo "S3 dryrun write validation running.." >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
touch $LOG_DIR/write.txt
aws s3 cp $LOG_DIR/write.txt "s3://$bucket_name" --dryrun >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1 ## directly add
S3_BKP_BKT to this
date >> $LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
if [ $? -ne 0 ]
then
echo "dryrun write failed in S3 bucket- $S3_BKP_BKT" >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
rm $LOG_DIR/write.txt
echo "DIFF Backup failure.dryrun write failed in S3 bucket-
$S3_BKP_BKT"| mailx -s "MYSQL $1 Backup Failure for component -
$component in $IP_ADDR" -a
"$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log" $recipients
exit 1
else
echo "dryrun write on S3 successful." >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
rm $LOG_DIR/write.txt
fi
date >> $LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
##### CHECK LSN DIRECTORY EXISTENCE ###########
echo "Checking if lsn directory exists $lsndir ..." >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
if [ ! -d $lsndir ]
then
echo "ERROR: the folder $lsndir does not exist" >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
date >> $LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log
2>&1
echo "Trying to create $lsndir" >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
mkdir -p $lsndir 2>>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log
if [ $? -ne 0 ]
then
echo "ERROR: Cannot create folder $lsndir" >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
echo "DIFF Backup failure.ERROR: Cannot create folder
$lsndir"| mailx -s "MYSQL $1 Backup Failure for component - $component in
$IP_ADDR" -a "$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log"
$recipients
exit 1
else
echo "$lsndir created successfully" >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
fi
date >> $LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log
2>&1
else
echo "Directory check successful." >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
fi
## FRAMING CURRENT DIFFERENTIAL NUMBER ###
DIFF_BKP_STS=`$MYSQL "select count(1) from dbbkpstatus where
ip_address='$IP_ADDR' and WEEK=$wk and ref_full_bkp='FULL$LFN' and
bkp_type='DIFFERENTIAL'"`
if [[ $DIFF_BKP_STS == 0 ]];
then
NUMBER=1;
else
LDN=`$MYSQL "select bkp_number from dbbkpstatus where
ip_address='$IP_ADDR' and WEEK=$wk and ref_full_bkp='FULL$LFN' and
bkp_type='DIFFERENTIAL' order by bkp_number desc limit 1"`
NUMBER=$(($LDN + 1))
fi
## GETTING LSN OF LAST SUCCESSFULL FULL BACKUP ##
TO_LSN=`$MYSQL "select to_lsn from dbbkpstatus where
ip_address='$IP_ADDR' and WEEK=$wk and bkp_type='FULL' and
bkp_number=$LFN and bkp_name='$LFNAME'"`
date >> $LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
$MYSQL "update dbbkpstatus set bkp_type='DIFFERENTIAL',
bkp_number=$NUMBER, ref_full_bkp='FULL$LFN',
local_bkp='$S3_BKP_PATH/${PRFX}_FULL${LFN}_DIFF$NUMBER' where
ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
echo "Initiating DIFFERENTIAL Backup Number $NUMBER" >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
xtrabackup --backup --stream=xbstream
--target-dir=$lsndir/${PRFX}_FULL${LFN}_DIFF${NUMBER}
--extra-lsndir=$lsndir/${PRFX}_FULL${LFN}_DIFF${NUMBER}
--incremental-lsn=$TO_LSN $ARGS 2>
$LOG_DIR/xtrabackup-${TS}_DIFF${NUMBER}.log |xbcloud put
$xbcloud_backup/${PRFX}_FULL${LFN}_DIFF$NUMBER --storage=s3
--s3-bucket=$bucket_name --s3-region='ap-south-1'
--s3-access-key=AKIAVOMR2TMUNZ3B56WL
--s3-secret-key=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
--parallel=1 2>
$LOG_DIR/xbcloud-${TS}-S3UPLOAD_DIFF${NUMBER}-stderr.log
stat=`echo ${PIPESTATUS[0]} ${PIPESTATUS[1]}` ## capture status of
xtrabackup|xbcloud
xtrabackup_status=`echo $stat|awk '{print $1}'`
xbcloud_status=`echo $stat|awk '{print $1}'`
date >> $LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
if [[ $xtrabackup_status -ne 0 || $xbcloud_status -ne 0 ]];
then
error_log=`tail -n 2
$LOG_DIR/xtrabackup-${TS}_DIFF${NUMBER}.log
$LOG_DIR/xbcloud-${TS}-S3UPLOAD_DIFF${NUMBER}-stderr.log`
mysql -u'open_user' -pF0rB@kPUse -h'10.100.64.53' inventory
-Bse "update dbbkpstatus set status='FAILED', end_date=now(),
remarks='$error_log' where ip_address='$IP_ADDR' and
bkp_name='$BKP_NAME'" >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
echo "Diff Backup failed at xtrabackup command. Check
attached logs"| mailx -s "MYSQL $1 Backup Failure for component -
$component in $IP_ADDR" -a
"$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log" -a
"$LOG_DIR/xtrabackup-${TS}_DIFF${NUMBER}.log" -a
"$LOG_DIR/xbcloud-${TS}-S3UPLOAD_DIFF${NUMBER}-stderr.log"
$recipients
cat $LOG_DIR/xtrabackup-${TS}_DIFF${NUMBER}.log >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
cat
$LOG_DIR/xbcloud-${TS}-S3UPLOAD_DIFF${NUMBER}-stderr.log >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
echo $error_log >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
rm $LOG_DIR/xtrabackup-${TS}_DIFF${NUMBER}.log
rm
$LOG_DIR/xbcloud-${TS}-S3UPLOAD_DIFF${NUMBER}-stderr.log
exit; exit;
fi
TO_LSN=`cat
$lsndir/${PRFX}_FULL${LFN}_DIFF${NUMBER}/xtrabackup_checkpoints |
grep to_lsn | awk -F" = " '{print$2}'`
FROM_LSN=`cat
$lsndir/${PRFX}_FULL${LFN}_DIFF${NUMBER}/xtrabackup_checkpoints |
grep from_lsn | awk -F" = " '{print$2}'`
if [[ ! -z `tail -1 $LOG_DIR/xtrabackup-${TS}_DIFF${NUMBER}.log | grep
'completed OK!'` && ! -z `tail -1
$LOG_DIR/xbcloud-${TS}-S3UPLOAD_DIFF${NUMBER}-stderr.log | grep
'Upload completed'` ]];
then
cat $LOG_DIR/xtrabackup-${TS}_DIFF${NUMBER}.log >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
cat
$LOG_DIR/xbcloud-${TS}-S3UPLOAD_DIFF${NUMBER}-stderr.log >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
rm $LOG_DIR/xtrabackup-${TS}_DIFF${NUMBER}.log
rm
$LOG_DIR/xbcloud-${TS}-S3UPLOAD_DIFF${NUMBER}-stderr.log
echo "DIFFERENTIAL Backup number $NUMBER & upload to s3
Completed, The preparation will be made when the restore is needed" >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
date >> $LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log
2>&1
echo "Differential $NUMBER COPY on S3 Completed ! " >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
$MYSQL "update dbbkpstatus set status='COMPLETED',
bkp_type='DIFFERENTIAL', bkp_number=$NUMBER, end_date=now(),
from_lsn=$FROM_LSN, to_lsn='$TO_LSN',
s3_bkp_path='$S3_BKP_BKT/$YEAR/WEEK$WEEK/$S3_BKP_DIR/$IP_AD
DR/${PRFX}_FULL${LFN}_INC$NUMBER',s3_bkp_dir='$S3_BKP_DIR'
where ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
else
error_log=`tail -n 2
$LOG_DIR/xtrabackup-${TS}_DIFF${NUMBER}.log
$LOG_DIR/xbcloud-${TS}-S3UPLOAD_DIFF${NUMBER}-stderr.log`
cat $LOG_DIR/xtrabackup-${TS}_DIFF${NUMBER}.log >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
cat
$LOG_DIR/xbcloud-${TS}-S3UPLOAD_DIFF${NUMBER}-stderr.log >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
rm $LOG_DIR/xtrabackup-${TS}_DIFF${NUMBER}.log
rm
$LOG_DIR/xbcloud-${TS}-S3UPLOAD_DIFF${NUMBER}-stderr.log
echo "Differential $NUMBER Failed !" >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
echo $error_log >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
echo "Diff Backup Failure in xtrabackup command.Review attached
log for details"| mailx -s "MYSQL $1 Backup Failure for component -
$component in $IP_ADDR" -a
"$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log" $recipients
$MYSQL "update dbbkpstatus set status='FAILED',
bkp_type='DIFFERENTIAL', bkp_number=$NUMBER, end_date=now() where
ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >>
$LOG_DIR/${PRFX}_FULL${LFN}_diff${NUMBER}.log 2>&1
fi
# rm -rf $BACKUP_DIR/xtra.sock
}
#######################################
##############MAIN#####################
#######################################
if [ $# -eq 0 ]
then
usage
exit 1
fi
if [ ! -d /home/dba/admin/xtrabackup_lsn ]
then
mkdir -p /home/dba/admin/xtrabackup_lsn;
#echo "ERROR: the folder '/dbschbackup/percona' does not exist"
#exit 1
fi
if [ ! -d /home/dba/admin/logs/BKPLOG ]
then
mkdir -p /home/dba/admin/logs/BKPLOG;
#echo "ERROR: the folder '/dbschbackup/percona' does not exist"
#exit 1
fi
recipients=
dt=`date '+%d%b%Y'`
wk=`date '+%W%Y'`
YEAR=`date '+%Y'`
MONT=`date '+%m'`
WEEK=`date '+%W'`
MYSQL="mysql -uopen_user -pF0rB@kPUse -h10.100.64.53 inventory -Bse"
mkdir -p /home/dba/admin/xtrabackup_lsn/WEEK$wk
mkdir -p /home/dba/admin/logs/BKPLOG/WEEK$wk
lsndir=/home/dba/admin/xtrabackup_lsn/WEEK$wk
LOG_DIR=/home/dba/admin/logs/BKPLOG/WEEK$wk
USER_ARGS=" --user=bkpusr --password=secret#987"
ARGS="$USER_ARGS --slave-info --kill-long-queries-timeout=3600
--kill-long-query-type=select --compress --compress-threads=1 --parallel=1
--tmpdir=$LOG_DIR --read-buffer-size=100M"
IP_ADDR=`/sbin/ifconfig | grep inet | grep -v 127.0.0.1 | grep -v '10.66' | grep
-v inet6 | awk '{print$2}'`
SFBN=`echo $IP_ADDR | cut -f2,3,4 -d'.' | sed "s/\.//g"`
BKP_NAME=`date '+%Y%m%d%H%M%S'$SFBN`
HST_NAME=`hostname -f`
S3_BKP_BKT=`mysql -u'open_user' -pF0rB@kPUse -h'10.100.64.53'
inventory -Bse "select s3_bkp_bucket from dbinfra where
ip_address='$IP_ADDR'"`
S3_BKP_DIR=`mysql -u'open_user' -pF0rB@kPUse -h'10.100.64.53'
inventory -Bse "select s3_bkp_dir from dbinfra where
ip_address='$IP_ADDR'"`
bucket_name=`echo $S3_BKP_BKT| sed 's/s3:\/\///g'`
S3_BKP_PATH=$S3_BKP_BKT/$YEAR/WEEK$WEEK/$S3_BKP_DIR/$IP_A
DDR
xbcloud_backup=$YEAR/WEEK$WEEK/$S3_BKP_DIR/$IP_ADDR
HST_ABBR=`hostname`
TS=`date '+%Y-%m-%d-%H-%M-%S'`
PRFX="Xtracmprsbkp_${HST_ABBR}_${TS}"
LWK="WEEK`date -d \"- 2 week\" '+%W%Y'`"
recipients=onlyjyoti@gmail.com
hostname=`hostname`
component=$(echo $hostname| awk -F '-' '{print $1"-"$2}')
trap "rm -f $lsndir/xtra.sock; sudo /bin/updatedb; exit" 0
#trap "$MYSQL \"update dbbkpstatus set status='FAILED', end_date=now()
where bkp_name='$BKP_NAME' and ip_address='$IP_ADDR'\"; exit" 1 2 3 8
9 14 15
$MYSQL "insert into
dbbkpstatus(ip_address,hostname,bkp_name,WEEK,MONTH,start_date,statu
s)
values('$IP_ADDR','$HST_NAME','$BKP_NAME',$wk,$MONT$YEAR,now(),'
started')" > /dev/null 2>&1
if [[ -d $lsndir/$LWK ]];
then
rm -rf $lsndir/$LWK
fi
if [[ -z $S3_BKP_BKT ]];
then
echo "Couldn't find s3_bkp_bucket for $IP_ADDR in Inventory.dbinfra" >>
$LOG_DIR/${PRFX}.log
$MYSQL "update dbbkpstatus set status='FAILED', start_date=now(),
end_date=now(), remarks='Could not find s3_bkp_bucket for $IP_ADDR in
Inventory.dbinfra' where ip_address='$IP_ADDR' and
bkp_name='$BKP_NAME'" >> $LOG_DIR/${PRFX}.log 2>&1
echo "Couldn't find s3_bkp_bucket for $IP_ADDR in Inventory.dbinfra" |
mailx -s "MYSQL $1 Backup Failure for component - $component in
$IP_ADDR" -a "$LOG_DIR/${PRFX}.log" $recipients
exit 1;
fi
if [[ -z $S3_BKP_DIR ]];
then
echo "Couldn't find s3_bkp_dir for $IP_ADDR in Inventory.dbinfra" >>
$LOG_DIR/${PRFX}.log
$MYSQL "update dbbkpstatus set status='FAILED', start_date=now(),
end_date=now(), remarks='Could not find s3_bkp_dir for $IP_ADDR in
Inventory.dbinfra' where ip_address='$IP_ADDR' and
bkp_name='$BKP_NAME'" >> $LOG_DIR/${PRFX}.log 2>&1
echo "Couldn't find s3_bkp_dir for $IP_ADDR in Inventory.dbinfra" | mailx
-s "MYSQL $1 Backup Failure in for component - $component $IP_ADDR" -a
"$LOG_DIR/${PRFX}.log" $recipients
exit 1;
fi
if [ -f $lsndir/xtra.sock ];
then
echo "Backup is already active on host, please wait for Completion. If no
active backup, please remove file $lsndir/xtra.sock and retry" >>
$LOG_DIR/${PRFX}.log
$MYSQL "update dbbkpstatus set status='FAILED', start_date=now(),
end_date=now(), remarks='Backup is already active on host, please wait for
Completion. If no active backup, please remove file $lsndir/xtra.sock and retry
' where ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >>
$LOG_DIR/${PRFX}.log 2>&1
echo "Backup is already active on host, please wait for Completion. If no
active backup, please remove file $lsndir/xtra.sock and retry" | mailx -s
"MYSQL $1 Backup Failure for component - $component in $IP_ADDR" -a
"$LOG_DIR/${PRFX}.log" $recipients
exit 1;
fi
touch $lsndir/xtra.sock;
case $1 in
"full")
full_backup
;;
"diff")
differential_backup
;;
"help")
usage
break
;;
*) echo "invalid option";;
esac
