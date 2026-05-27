# Oracle-Database-Convert-non-CDB-to-CDB
Database release from Version 19C oracle recommend to use with pluggable database.

I.Environment
-----------------------------------------------------------------
	Platform		: Linux x86_64                              
	Server Name		: RAC1.RAJASEKHAR.COM, IP: 10.xx.xx.21    
	DB Version		: Oracle 19.20.0.0, File system: Normal     
	Source NON-CDB  : DESTDB  

	
	Target CDB 		: CDB1                                     
	Target PDB  	: DESTDB                         	
	Oracle Home Path        : /u02/app/oracle/product/19.3.0/dbhome_1  
-----------------------------------------------------------------

II. On Sorce Database instance 

	#1 .Shutdown NON-CDB (DESTDB)
			SQL> select name, open_mode, cdb from v$database;
			
			NAME      OPEN_MODE            CDB
			--------- -------------------- ---
			DESTDB    READ WRITE           NO <------------------------
			
			SQL>

		-optional

		SQL> create user convert_user identified by welcome1 default tablespace users temporary tablespace temp;
		SQL> grant connect,resource to convert_user;
		SQL> alter user convert_user quota unlimited on users;
		SQL> conn convert_user/welcome1;
		SQL> show user
		USER is "CONVERT_USER"
		
		SQL> create table position(Name varchar2(10), Role varchar2(10));
		SQL> insert into position values ('&a','&b');
		Enter value for a: Jack
		Enter value for b: DBA
		old   1: insert into position values ('&a','&b')
		new   1: insert into position values ('Jack','DBA')
		
		1 row created.
	
		SQL> /
		Enter value for a: Jack01
		Enter value for b: DBA
		old   1: insert into position values ('&a','&b')
		new   1: insert into position values ('Jack01','DBA')
		
		1 row created.
		
		SQL> /
		Enter value for a: Jack02
		Enter value for b: DBA
		old   1: insert into position values ('&a','&b')
		new   1: insert into position values ('Jack02','DBA')
		
		1 row created.
		
		SQL> /
		Enter value for a: Jack03
		Enter value for b: DBA
		old   1: insert into position values ('&a','&b')
		new   1: insert into position values ('Jack03','DBA')
		
		1 row created.
		
		SQL>
		SQL> commit;
		
		Commit complete.
	
		SQL> select * from position;
		
		NAME       ROLE
		---------- ----------
		Jack     DBA
		Jack01   DBA
		Jack02   DBA
		Jack03   DBA
		
		SQL>
		-------------------------------------------------------------------------
		SQL> CONN / AS SYSDBA
		Connected.
		SQL> show user;
		USER is "SYS"
		SQL> ALTER SYSTEM FLUSH BUFFER_CACHE;
		
		System altered.
		
		SQL> select name from v$datafile;
	
		NAME
		--------------------------------------------------------------------------------
		/u02/app/oracle/oradata/DESTDB/datafile/o1_mf_system_lpnddh21_.dbf
		/u02/app/oracle/oradata/DESTDB/datafile/o1_mf_sysaux_lpndfl75_.dbf
		/u02/app/oracle/oradata/DESTDB/datafile/o1_mf_undotbs1_lpndgcbb_.dbf
		/u02/app/oracle/oradata/DESTDB/datafile/TBS1_5.dbf
		/u02/app/oracle/oradata/DESTDB/datafile/o1_mf_users_lpndgdf0_.dbf
		
		SQL> shut immediate;
		Database closed.
		Database dismounted.
		ORACLE instance shut down.
		SQL>

	#2. Open Non-CDB in read only (this is for consistency)

		SQL> STARTUP MOUNT EXCLUSIVE;
		ORACLE instance started.
		
		Total System Global Area 3204445168 bytes
		Fixed Size                  8930288 bytes
		Variable Size             889192448 bytes
		Database Buffers         2298478592 bytes
		Redo Buffers                7843840 bytes
		Database mounted.
		SQL>
		SQL> alter database open read only; <-----
		
		Database altered.
		
		SQL> select name, open_mode, cdb from v$database;
		
		NAME      OPEN_MODE            CDB
		--------- -------------------- ---
		DESTDB    READ ONLY            NO <----- status of non-cdb database instance
		
		SQL>

	#3. Run DBMS_PDB.DESCRIBE to create an XML file

		mkdir -p /home/oracle/convertCDB
		SQL> 
		BEGIN
		DBMS_PDB.DESCRIBE(pdb_descr_file => '/home/oracle/convertCDB/noncdb19c.xml');
		END;
		/ 
	
		$ ls -l
		-rw-r--r--. 1 oracle oinstall 7959 Feb  2 11:28 noncdb19c.xml <---- created file


	#4. Shutdown the Non-CDB

		SQL> select name, open_mode, cdb from v$database;
		
		NAME      OPEN_MODE            CDB
		--------- -------------------- ---
		DESTDB    READ ONLY            NO <-----
		
		SQL> SHUTDOWN IMMEDIATE;
		Database closed.
		Database dismounted.
		ORACLE instance shut down.
		SQL>


II. On Target database instance

	#1. Connect to Target DB instance

		--- profile 
		$ cat cdb_instance.env
		export TMP=/tmp
		export TMPDIR=$TMP
		export ORACLE_BASE=/u02/app/oracle
		export DB_HOME=$ORACLE_BASE/product/19.3.0/dbhome_1
		export ORACLE_HOME=$DB_HOME
		export ORACLE_SID=cdb1
		export ORACLE_TERM=xterm
		export BASE_PATH=/usr/sbin:$PATH
		export MW_HOME=/u02/app/oracle/middleware
		export JAVA_HOME=/u02/app/oracle/jdk1.8.0_202
		export PATH=$ORACLE_HOME/bin:$JAVA_HOME/bin:$BASE_PATH
		export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
		export CLASSPATH=$ORACLE_HOME/JRE:$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib
		alias sql='sqlplus / as sysdba'

		SQL> select name, open_mode, cdb from v$database;
		
		NAME      OPEN_MODE            CDB
		--------- -------------------- ---
		CDB1      READ WRITE           YES <------
		
		SQL>

	#2.Check whether non-cdb (DESTDB) can be plugged into CDB(CDB1)
		SQL> 
		
		SET SERVEROUTPUT ON
		DECLARE
		compatible CONSTANT VARCHAR2(3) :=
		CASE DBMS_PDB.CHECK_PLUG_COMPATIBILITY(
		pdb_descr_file => '/home/oracle/convertCDB/noncdb19c.xml',
		pdb_name => 'NONCDB12')
		WHEN TRUE THEN 'YES'
		ELSE 'NO'
		END;
		BEGIN
		DBMS_OUTPUT.PUT_LINE(compatible);
		END;
		/
		
		SQL>   2    3    4    5    6    7    8    9   10   11   12
		
		YES  									<------------------- result after validated
		
		PL/SQL procedure successfully completed.
		
		SQL>

	#3.Create target directory for datafiles

		SQL> !mkdir -p /u02/app/oracle/oradata/CDB1/DESTDB

	#4.Plug-in Non-CDB (DESTDB) as PDB (DESTDB) into CDB1
		- plugging the database in to a CDB on the same server with COPY clause and hence using  FILE_NAME_CONVERT.

		SQL> CREATE PLUGGABLE DATABASE DESTDB USING '/home/oracle/convertCDB/noncdb19c.xml' COPY file_name_convert=('/u02/app/oracle/oradata/DESTDB','/u02/app/oracle/oradata/CDB1/DESTDB');

				******************************************************************************************************************************************************************************************
				* Known Issue : <-----------
					* SQL> CREATE PLUGGABLE DATABASE DESTDB USING '/home/oracle/convertCDB/noncdb19c.xml' COPY file_name_convert=('/u02/app/oracle/oradata/DESTDB','/u02/app/oracle/oradata/CDB1/DESTDB');
					* CREATE PLUGGABLE DATABASE DESTDB USING '/home/oracle/convertCDB/noncdb19c.xml' COPY file_name_convert=('/u02/app/oracle/oradata/DESTDB','/u02/app/oracle/oradata/CDB1/DESTDB')
					* *
					* ERROR at line 1:
					* ORA-01276: Cannot add file
					* /u02/app/oracle/oradata/CDB1/DESTDB/datafile/o1_mf_system_lpnddh21_.dbf.  File
					* has an Oracle Managed Files file name.  <------------------------------------------------ failed with OMF 
					* 
					* 
					* SQL>

				- solution no need to mention path of datafile 
				
					SQL> CREATE PLUGGABLE DATABASE DESTDB USING '/home/oracle/convertCDB/noncdb19c.xml';
					
					Pluggable database created.
					
					SQL>

				*********************************************************************************************************************************************************************************************


	#4.Verify newly created PDB DESTDB on CDB1

		SQL> select name, open_mode, cdb from v$database;
		
		NAME      OPEN_MODE            CDB
		--------- -------------------- ---
		CDB1      READ WRITE           YES <-------------
		
		SQL> 
		col pdb_name for a30
		SELECT pdb_name , status from cdb_pdbs;
		SQL>
		
		PDB_NAME                       STATUS
		------------------------------ ----------
		DESTDB                         NEW  <-----------  new instance as pdb
		PDB$SEED                       NORMAL
		
		SQL>

		--------------------- OR -------------------------------------------
		SQL> select pdb_name, status from dba_pdbs;
		
		PDB_NAME                       STATUS
		------------------------------ ----------
		DESTDB                         NEW <------------ 
		PDB$SEED                       NORMAL
		
		SQL>
		SQL> col name for a30
		select name, open_mode from v$pdbs;SQL>
		
		NAME                           OPEN_MODE
		------------------------------ ----------
		PDB$SEED                       READ ONLY
		DESTDB                         MOUNTED <-----------------
		
		SQL>

		--check that datafiles copied to new location 
		SQL> !ls -ltr /u02/app/oracle/oradata/CDB1/0B78A8B7958D6533E0631EC9760A2F5B/datafile
		total 8770672
		-rw-r-----. 1 oracle oinstall  145760256 Feb  2 11:53 o1_mf_temp_lvrx8ck5_.dbf
		-rw-r-----. 1 oracle oinstall    5251072 Feb  2 11:53 o1_mf_users_lvrx8ck6_.dbf
		-rw-r-----. 1 oracle oinstall  786440192 Feb  2 11:53 o1_mf_undotbs1_lvrx8ck4_.dbf
		-rw-r-----. 1 oracle oinstall 5368717312 Feb  2 11:53 o1_mf_tbs1_lvrx8ck7_.dbf
		-rw-r-----. 1 oracle oinstall 1163927552 Feb  2 11:53 o1_mf_system_lvrx8cjd_.dbf
		-rw-r-----. 1 oracle oinstall 1656758272 Feb  2 11:53 o1_mf_sysaux_lvrx8ck3_.dbf


	#5.Run noncdb_to_pdb.sql script on new PDB (DESTDB)
		- found : $ORACLE_HOME/rdbms/admin/noncdb_to_pdb.sql

		SQL> select name, open_mode, cdb from v$database;
		
		NAME                           OPEN_MODE            CDB
		------------------------------ -------------------- ---
		CDB1                           READ WRITE           YES

		SQL> ALTER SESSION SET CONTAINER=DESTDB;
		
		Session altered.
		
		SQL> show con_name
		
		CON_NAME
		------------------------------
		DESTDB
		SQL> show con_id

		CON_ID
		------------------------------
		3

		
		SQL> @$ORACLE_HOME/rdbms/admin/noncdb_to_pdb.sql;
		SQL> SET FEEDBACK 1
		SQL> SET NUMWIDTH 10
		SQL> SET LINESIZE 80
		SQL> SET TRIMSPOOL ON
		SQL> SET TAB OFF
		SQL> SET PAGESIZE 100
		SQL> SET VERIFY OFF
		SQL>
		SQL> WHENEVER SQLERROR EXIT;
		SQL>
		SQL> DOC
		DOC>#######################################################################
		DOC>#######################################################################
		DOC>   The following statement will cause an "ORA-01403: no data found"
		DOC>   error if we're not in a PDB.
		DOC>   This script is intended to be run right after plugin of a PDB,
		DOC>   while inside the PDB.
		DOC>#######################################################################
		DOC>#######################################################################
		DOC>#
		=================================================================================================
		log file put in 
		=================================================================================================
		
		PL/SQL procedure successfully completed.
		
		SQL>
		SQL> WHENEVER SQLERROR CONTINUE;
		SQL>

	#6. Verify table PDB_PLUG_IN_VIOLATIONS
		SQL> 
		col cause for a10
		col name for a10
		col message for a35 word_wrapped
		select name,cause,type,message,status from PDB_PLUG_IN_VIOLATIONS where name='DESTDB';
		
		NAME       CAUSE      TYPE      MESSAGE                             STATUS
		---------- ---------- --------- ----------------------------------- ---------
		DESTDB     Parameter  WARNING   CDB parameter sga_target mismatch:  RESOLVED <-------
		                                Previous 3056M Current 2304M
		
		DESTDB     Parameter  WARNING   CDB parameter pga_aggregate_target  RESOLVED <-------
		                                mismatch: Previous 1015M Current
		                                767M
		
		DESTDB     Non-CDB to ERROR     PDB plugged in is a non-CDB,        PENDING
		            PDB                 requires noncdb_to_pdb.sql be run.
		
		
		SQL>

		SQL> show pdbs;
		
		    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
		---------- ------------------------------ ---------- ----------
		         2 PDB$SEED                       READ ONLY  NO
		         3 DESTDB                         MOUNTED
						 
		SQL> alter session set container=DESTDB;
		Session altered.
		
		SQL> show con_name
		CON_NAME
		------------------------------
		DESTDB
		
		SQL> alter database open;
		Database altered.
		
		SQL> show pdbs;
		    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
		---------- ------------------------------ ---------- ----------
		         3 DESTDB                         READ WRITE NO  <----------
		SQL>


		$ sqlplus / as sysdba
		SQL> show con_name
		CON_NAME
		------------------------------
		CDB$ROOT

		SQL> select name, open_mode, cdb from v$database;
		NAME      OPEN_MODE            CDB
		--------- -------------------- ---
		CDB1      READ WRITE           YES


		col pdb_name for a30
		SELECT pdb_name , status from cdb_pdbs;
		PDB_NAME                       STATUS
		------------------------------ ----------
		DESTDB                         NORMAL <------
		PDB$SEED                       NORMAL

		SQL> 
		col cause for a10
		col name for a10
		col message for a35 word_wrapped
		select name,cause,type,message,status from PDB_PLUG_IN_VIOLATIONS where name='DESTDB';
		
		NAME       CAUSE      TYPE      MESSAGE                             STATUS
		---------- ---------- --------- ----------------------------------- ---------
		DESTDB     Parameter  WARNING   CDB parameter sga_target mismatch:  RESOLVED  <-------
		                                Previous 3056M Current 2304M
		
		DESTDB     Parameter  WARNING   CDB parameter pga_aggregate_target  RESOLVED <--------
		                                mismatch: Previous 1015M Current
		                                767M
		
		DESTDB     Non-CDB to ERROR     PDB plugged in is a non-CDB,        RESOLVED <--------
		            PDB                 requires noncdb_to_pdb.sql be run.


SQL> alter session set container=DESTDB;
SQL> select * from CONVERT_USER.position;

NAME     ROLE
------	 ----------
Jack     DBA <------
Jack01   DBA <------
Jack02   DBA <------
Jack03   DBA <------

SQL>
