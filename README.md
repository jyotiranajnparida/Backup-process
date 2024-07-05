# Backup-process

#set -e #stops execution if a variable is not set
#set -u #stop execution if something goes wrong
set -x

usage() {
        echo "usage: $(basename $0) [option]"
        echo "option=full: do a full backup using innobackupex"
        echo "option=incremental: do a incremental backup"
        echo "option=help: show this help"
}

full_backup() {

	### CHECK COUNT OF FULL BACKUPS IN CURRENT WEEK ################
        FULL_BKP_STS=`$MYSQL "select count(1) from dbbkpstatus where ip_address='$IP_ADDR' and WEEK=$wk and bkp_type='FULL';"`

        if [ $FULL_BKP_STS -eq 0 ];
        then
                NUMBER=1;
        else
                PFN=`$MYSQL "select bkp_number from dbbkpstatus where ip_address='$IP_ADDR' and bkp_type='FULL' and WEEK=$wk order by bkp_number desc limit 1"`
                NUMBER=$(($PFN + 1))
        fi

        date > $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1

	##### CHECK BACKUP DIRECTORY EXISTANCE ###########
        if [ ! -d $BACKUP_DIR ]
        then
                echo "ERROR: the folder $BACKUP_DIR does not exist" >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1
                #rm -rf $BACKUP_DIR/xtra.sock
                exit 1
        fi

	##### INITIATING FULL BACKUP #######################
        
        $MYSQL "update dbbkpstatus set bkp_type='FULL',bkp_number=$NUMBER where ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1

        echo "Initialting full backup Number ${NUMBER}..." >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1
        innobackupex $ARGS $BACKUP_DIR/${PRFX}_FULL${NUMBER} >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1

        if [[ $? -ne 0 ]];
        then
                error_log=`tail -2 $LOG_DIR/${PRFX}_FULL${NUMBER}.log`
                mysql -u'user_create' -p'Erstfxccc' -h'10.X.X.X' inventory -Bse "update dbbkpstatus set status='FAILED' , end_date=now(), remarks='$error_log' where ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1
                exit; exit;
        fi

        if [[ ! -z `tail -1 ${LOG_DIR}/${PRFX}_FULL${NUMBER}.log | grep 'completed OK!'` ]];
        then
                echo "Percona Full Backup Completed" >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1
                date >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1
                echo "Intiating S3 Upload..." >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1
                aws s3 cp $BACKUP_DIR/${PRFX}_FULL${NUMBER} $S3_BKP_BKT/$YEAR/WEEK$WEEK/$S3_BKP_DIR/$IP_ADDR/${PRFX}_FULL${NUMBER}/ --recursive --quiet >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1

                if [[ $? -ne 0 ]];
                then
                        mysql -u'user_create' -p'Erstfxccc' -h'10.X.X.X' inventory -Bse "update dbbkpstatus set status='FAILED', end_date=now(), remarks='Backup Failed while uploading to S3.' where ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1
                        echo "Backup Failed while uploading to S3." >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1
                        exit; exit;
                fi

                date >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1
                echo "S3 Upload Completed !" >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1
                TO_LSN=`cat $BACKUP_DIR/${PRFX}_FULL${NUMBER}/xtrabackup_checkpoints | grep to_lsn | awk -F" = " '{print$2}'`
		FROM_LSN=`cat $BACKUP_DIR/${PRFX}_FULL${NUMBER}/xtrabackup_checkpoints | grep from_lsn | awk -F" = " '{print$2}'`
                $MYSQL "update dbbkpstatus set status='COMPLETED', bkp_type='FULL', bkp_number=${NUMBER}, from_lsn=$FROM_LSN, to_lsn='$TO_LSN', end_date=now(), s3_bkp_path='$S3_BKP_BKT/$YEAR/WEEK$WEEK/$S3_BKP_DIR/$IP_ADDR/${PRFX}_FULL${NUMBER}',s3_bkp_dir='$S3_BKP_DIR'  where ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1
		
		date >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1

		transac=`$MYSQL "select is_transactional from dbinfra where ip_address='$IP_ADDR'"`
		if [[ $transac == 'Yes' ]];
		then
			MNTH_FULL_BKP_CNT=`$MYSQL "select count(*) from dbbkpstatus where ip_address='$IP_ADDR' and bkp_type='FULL' and MONTH=$MONT$YEAR and status='COMPLETED'"`
			if [[ $MNTH_FULL_BKP_CNT == 1 ]]
			then
				perstbkt=`$MYSQL "select s3_persistant_bkp_bucket from dbinfra where ip_address='$IP_ADDR'"`
                                cmpfullbkppath=`$MYSQL "select s3_bkp_path from dbbkpstatus where ip_address='$IP_ADDR' and bkp_type='FULL' and MONTH=$MONT$YEAR and status='COMPLETED'"`
				cmpfullbkpname=`echo $cmpfullbkppath | rev | cut -f1 -d'/' | rev`
				$MYSQL "insert into dbbkpstatus_mnthly(ip_address,hostname,status,MONTH,s3_bkp_path,start_date) values('$IP_ADDR','$HST_NAME','STARTED','$MONT$YEAR','${perstbkt}/$YEAR/MONTH${MONT}/$S3_BKP_DIR/$cmpfullbkpname',now())"
				aws s3 sync $cmpfullbkppath/ ${perstbkt}/$YEAR/MONTH${MONT}/$S3_BKP_DIR/$cmpfullbkpname/ >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1
				if [[ $? -eq 0 ]];
				then
					$MYSQL "update dbbkpstatus_mnthly set status='COMPLETED',end_date=now() where ip_address='$IP_ADDR' and MONTH=$MONT$YEAR;"
				else
					$MYSQL "update dbbkpstatus_mnthly set status='FAILED',end_date=now() where ip_address='$IP_ADDR' and MONTH=$MONT$YEAR;"
				fi
			else
				echo "No more Persistant backup uploads required this month." >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1
			fi
			
		fi
	else
                date >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1
                echo "Percona Full Backup Number ${NUMBER} Failed" >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1
                mysql -u'user_create' -p'Erstfxccc' -h'10.X.X.X' inventory -Bse "update dbbkpstatus set status='FAILED', bkp_type='FULL' and bkp_number=${NUMBER}, end_date=now() where ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >> $LOG_DIR/${PRFX}_FULL${NUMBER}.log 2>&1
        fi
        #rm -rf $BACKUP_DIR/xtra.sock
}

incremental_backup()
{

        FULL_BKP_STS=`$MYSQL "select count(1) from dbbkpstatus where ip_address='$IP_ADDR' and WEEK=$wk and bkp_type='FULL' and status='COMPLETED'"`

        if [[ $FULL_BKP_STS == 0 ]];
        then
                echo "ERROR: No Succesfull Full backup has been done before this week. aborting" >> $LOG_DIR/${PRFX}.log 2>&1
                $MYSQL "update dbbkpstatus set status='FAILED', bkp_type='INCREMENTAL', start_date=now(), end_date=now(), remarks='No Succesfull Full backup has been done before this week.' where ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >> $LOG_DIR/${PRFX}.log 2>&1
                exit;
        else
                LFN=`$MYSQL "select bkp_number from dbbkpstatus where ip_address='$IP_ADDR' and bkp_type='FULL' and WEEK=$wk and status='COMPLETED' order by bkp_number desc limit 1"`
                LFNAME=`$MYSQL "select bkp_name from dbbkpstatus where ip_address='$IP_ADDR' and bkp_type='FULL' and WEEK=$wk and status='COMPLETED' order by bkp_number desc limit 1"`
        fi


        ## FRAMING CURRENT INCREMENTAL NUMBER ###

        INC_BKP_STS=`$MYSQL "select count(1) from dbbkpstatus where ip_address='$IP_ADDR' and WEEK=$wk and ref_full_bkp='FULL$LFN' and bkp_type='INCREMENTAL'"`

        if [[ $INC_BKP_STS == 0 ]];
        then
                NUMBER=1;
        else
                LIN=`$MYSQL "select bkp_number from dbbkpstatus where ip_address='$IP_ADDR' and WEEK=$wk and ref_full_bkp='FULL$LFN' and bkp_type='INCREMENTAL' order by bkp_number desc limit 1"`

                NUMBER=$(($LIN + 1))
        fi

        ## GETTING LSN OF LAST SUCCESSFULL BACKUP ##

        INC_CMP_BKP=`$MYSQL "select count(1) from dbbkpstatus where ip_address='$IP_ADDR' and WEEK=$wk and ref_full_bkp='FULL$LFN' and bkp_type='INCREMENTAL' and status='COMPLETED'"`

        if [[ $INC_CMP_BKP == 0 ]];
        then
                TO_LSN=`$MYSQL "select to_lsn from dbbkpstatus where ip_address='$IP_ADDR' and WEEK=$wk and bkp_type='FULL' and bkp_number=$LFN and bkp_name='$LFNAME'"`
        else
                TO_LSN=`$MYSQL "select to_lsn from dbbkpstatus where ip_address='$IP_ADDR' and WEEK=$wk and ref_full_bkp='FULL$LFN' and bkp_type='INCREMENTAL' and status='COMPLETED' order by bkp_number desc limit 1"`
        fi

        date >> $LOG_DIR/${PRFX}_FULL${LFN}_INC$NUMBER.log 2>&1

        $MYSQL "update dbbkpstatus set bkp_type='INCREMENTAL', bkp_number=$NUMBER, ref_full_bkp='FULL$LFN' where ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >> $LOG_DIR/${PRFX}_FULL${LFN}_INC$NUMBER.log 2>&1

        echo "Initiating Incremental Backup Number $NUMBER" >> $LOG_DIR/${PRFX}_FULL${LFN}_INC$NUMBER.log 2>&1
        innobackupex $ARGS --incremental $BACKUP_DIR/${PRFX}_FULL${LFN}_INC$NUMBER --incremental-lsn=$TO_LSN >> $LOG_DIR/${PRFX}_FULL${LFN}_INC$NUMBER.log 2>&1

                if [[ $? -ne 0 ]];
                then
                        error_log=`tail -2 $LOG_DIR/${PRFX}_FULL${LFN}_INC$NUMBER.log`
                        mysql -u'user_create' -p'Erstfxccc' -h'10.X.X.X' inventory -Bse "update dbbkpstatus set status='FAILED', end_date=now(), remarks='$error_log' where ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >> $LOG_DIR/${PRFX}_FULL${LFN}_INC$NUMBER.log 2>&1
                        exit; exit;
                fi

        TO_LSN=`cat $BACKUP_DIR/${PRFX}_FULL${LFN}_INC${NUMBER}/xtrabackup_checkpoints | grep to_lsn | awk -F" = " '{print$2}'`
	FROM_LSN=`cat $BACKUP_DIR/${PRFX}_FULL${LFN}_INC${NUMBER}/xtrabackup_checkpoints | grep from_lsn | awk -F" = " '{print$2}'`

        if [[ ! -z `tail -1 ${LOG_DIR}/${PRFX}_FULL${LFN}_INC${NUMBER}.log | grep 'completed OK!'` ]];
        then
                echo "Incremental Backup number $NUMBER Completed !, The preparation will be made when the restore is needed" >> $LOG_DIR/${PRFX}_FULL${LFN}_INC$NUMBER.log 2>&1
                date >> $LOG_DIR/${PRFX}_FULL${LFN}_INC$NUMBER.log 2>&1
                aws s3 cp $BACKUP_DIR/${PRFX}_FULL${LFN}_INC$NUMBER $S3_BKP_BKT/$YEAR/WEEK$WEEK/$S3_BKP_DIR/$IP_ADDR/${PRFX}_FULL${LFN}_INC$NUMBER/ --recursive --quiet >> $LOG_DIR/${PRFX}_FULL${LFN}_INC$NUMBER.log 2>&1

                if [[ $? -ne 0 ]];
                then
                        mysql -u'user_create' -p'Erstfxccc' -h'10.X.X.X' inventory -Bse "update dbbkpstatus set status='FAILED', end_date=now(), remarks='Backup Failed while uploading to S3.' where ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >> $LOG_DIR/${PRFX}_FULL${LFN}_INC$NUMBER.log 2>&1
                        echo "Backup Failed while uploading to S3." >> $LOG_DIR/${PRFX}_FULL${LFN}_INC$NUMBER.log 2>&1
                        exit; exit;
                fi

                echo "Incremental $NUMBER COPY on S3 Completed ! " >> $LOG_DIR/${PRFX}_FULL${LFN}_INC$NUMBER.log 2>&1
                $MYSQL "update dbbkpstatus set status='COMPLETED', bkp_type='INCREMENTAL', bkp_number=$NUMBER, end_date=now(), from_lsn=$FROM_LSN, to_lsn='$TO_LSN', s3_bkp_path='$S3_BKP_BKT/$YEAR/WEEK$WEEK/$S3_BKP_DIR/$IP_ADDR/${PRFX}_FULL${LFN}_INC$NUMBER',s3_bkp_dir='$S3_BKP_DIR' where ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >> $LOG_DIR/${PRFX}_FULL${LFN}_INC$NUMBER.log 2>&1
        else
                echo "Incremental $NUMBER Failed !" >> $LOG_DIR/${PRFX}_FULL${LFN}_INC$NUMBER.log 2>&1
                $MYSQL "update dbbkpstatus set status='FAILED', bkp_type='INCREMENTAL', bkp_number=$NUMBER, end_date=now() where ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >> $LOG_DIR/${PRFX}_FULL${LFN}_INC$NUMBER.log 2>&1
        fi
#        rm -rf $BACKUP_DIR/xtra.sock
}

#######################################
##############MAIN#####################
#######################################

if [ $# -eq 0 ]
then
usage
exit 1
fi

if [ ! -d /data/mysqlbackup/DB_BKP ]
then
        mkdir -p /data/mysqlbackup/DB_BKP;
    #echo "ERROR: the folder '/dbschbackup/percona' does not exist"
    #exit 1
fi

if [ ! -d /home/dba/admin/logs/BKPLOG ]
then
        mkdir -p /home/dba/admin/logs/BKPLOG;
    #echo "ERROR: the folder '/dbschbackup/percona' does not exist"
    #exit 1
fi


dt=`date '+%d%b%Y'`
wk=`date '+%W%Y'`
YEAR=`date '+%Y'`
MONT=`date '+%m'`
WEEK=`date '+%W'`
MYSQL="mysql -uuser_create -pErstfxccc -h10.X.X.X inventory -Bse"

mkdir -p /data/mysqlbackup/DB_BKP/WEEK$wk
mkdir -p /home/dba/admin/logs/BKPLOG/WEEK$wk

BACKUP_DIR=/data/mysqlbackup/DB_BKP/WEEK$wk
LOG_DIR=/home/dba/admin/logs/BKPLOG/WEEK$wk

USER_ARGS=" --user=bkpusr --password=secret#987"
ARGS="$USER_ARGS --no-timestamp --slave-info --kill-long-queries-timeout=3600 --kill-long-query-type=select --compress --compress-threads=1 --parallel=1 --tmpdir=$LOG_DIR"
#ARGS="$USER_ARGS --no-timestamp --slave-info --tmpdir=$LOG_DIR"


IP_ADDR=`/sbin/ifconfig | grep inet | grep -v 127.0.0.1 | grep -v '10.66' | grep -v inet6 | awk '{print$2}'`
SFBN=`echo $IP_ADDR | cut -f2,3,4 -d'.' | sed "s/\.//g"`
BKP_NAME=`date '+%Y%m%d%H%M%S'$SFBN`
HST_NAME=`hostname -f`
S3_BKP_BKT=`mysql -u'user_create' -p'Erstfxccc' -h'10.X.X.X' inventory -Bse "select s3_bkp_bucket from dbinfra where ip_address='$IP_ADDR'"`
S3_BKP_DIR=`mysql -u'user_create' -p'Erstfxccc' -h'10.X.X.X' inventory -Bse "select s3_bkp_dir from dbinfra where ip_address='$IP_ADDR'"`

HST_ABBR=`hostname`
TS=`date '+%Y-%m-%d-%H-%M-%S'`
#PRFX="Xtracmprsbkp_${HST_ABBR}_${BKP_NAME}"
PRFX="Xtracmprsbkp_${HST_ABBR}_${TS}"
LWK="WEEK`date -d \"- 1 week\" '+%W%Y'`"


trap "rm -f $BACKUP_DIR/xtra.sock; sudo /bin/updatedb; exit" 0
#trap "$MYSQL \"update dbbkpstatus set status='FAILED', end_date=now() where bkp_name='$BKP_NAME' and ip_address='$IP_ADDR'\"; exit" 1 2 3 8 9 14 15

$MYSQL "insert into dbbkpstatus(ip_address,hostname,bkp_name,WEEK,MONTH,start_date,status) values('$IP_ADDR','$HST_NAME','$BKP_NAME',$wk,$MONT$YEAR,now(),'started')" > /dev/null 2>&1


if [[ -d /data/mysqlbackup/DB_BKP/$LWK ]];
then
        rm -rf /data/mysqlbackup/DB_BKP/$LWK
fi

if ls /data/mysqlbackup/DB_BKP/db_bkp_xtrabkp* 1> /dev/null 2>&1;
then
        rm -rf /data/mysqlbackup/DB_BKP/db_bkp_xtrabkp*
fi

if [[ -z $S3_BKP_BKT ]];
then
    echo "Couldn't find s3_bkp_bucket for $IP_ADDR in Inventory.dbinfra" >> $LOG_DIR/${PRFX}.log
        $MYSQL "update dbbkpstatus set status='FAILED', start_date=now(), end_date=now(), remarks='Could not find s3_bkp_bucket for $IP_ADDR in Inventory.dbinfra' where ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >> $LOG_DIR/${PRFX}.log 2>&1
        exit 1;
fi

if [[ -z $S3_BKP_DIR ]];
then
        echo "Couldn't find s3_bkp_dir for $IP_ADDR in Inventory.dbinfra" >> $LOG_DIR/${PRFX}.log
        $MYSQL "update dbbkpstatus set status='FAILED', start_date=now(), end_date=now(), remarks='Could not find s3_bkp_dir for $IP_ADDR in Inventory.dbinfra' where ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >> $LOG_DIR/${PRFX}.log 2>&1
        exit 1;
fi

if [ -f $BACKUP_DIR/xtra.sock ];
then
        echo "Backup is already active on host, please wait for Completion. If no active backup, please remove file $BACKUP_DIR/xtra.sock and retry" >> $LOG_DIR/${PRFX}.log
        $MYSQL "update dbbkpstatus set status='FAILED', start_date=now(), end_date=now(), remarks='Backup is already active on host, please wait for Completion. If no active backup, please remove file $BACKUP_DIR/xtra.sock and retry ' where ip_address='$IP_ADDR' and bkp_name='$BKP_NAME'" >> $LOG_DIR/${PRFX}.log 2>&1
        exit 1;
fi


touch $BACKUP_DIR/xtra.sock;

    case $1 in
        "full")
            full_backup
            ;;
        "incremental")
        incremental_backup
            ;;
        "help")
            usage
            break
            ;;
        *) echo "invalid option";;
    esac

