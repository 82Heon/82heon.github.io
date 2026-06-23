---
title: xtts_test_case (AIX to LINUX)
parent: Oracle
nav_order: 1
---

{:toc}

# 개요
## XTTS?
- 데이터베이스를 마이그레이션 하기 위한 하나의 방법
- 테이블스페이스 단위로 데이터파일을 복사하여 데이터베이스를 이관
- 타겟의 데이터베이스를 유지한 상태에서 소스의 특정 테이블스페이스만 이관 시 유용
- 플랫폼에 따라 데이터 파일의 복사를 통해 마이그레이션을 수행할 수 있으나 소스와 타겟의 Endian 차이가 발생할 경우 단순 복사를 통한 TTS 가 불가능
- 이를 해결하기 위해 각 데이터파일의 Endian 포맷을 타겟의 Endian 포맷으로 Convert 후 TTS 를 수행하는 방식을 XTTS 라고 함.

## XTTS  가능 플랫폼 리스트 확인
- 현재 해당 사용하는 데이터베이스가 어떠한 플랫폼으로 변경할 수 있는 지 확인하기 위해서는 `V$TRANSPORTABLE_PLATFORM` 뷰를 참조한다.
    
    ```sql
    COL PLATFORM_NAME FOR A40
    SET PAGES 500
    SELECT PLATFORM_ID, PLATFORM_NAME, ENDIAN_FORMAT
      FROM V$TRANSPORTABLE_PLATFORM
    ORDER BY PLATFORM_ID;
    
    PLATFORM_ID PLATFORM_NAME                            ENDIAN_FORMAT
    ----------- ---------------------------------------- --------------
              1 Solaris[tm] OE (32-bit)                  Big
              2 Solaris[tm] OE (64-bit)                  Big
              3 HP-UX (64-bit)                           Big
              4 HP-UX IA (64-bit)                        Big
              5 HP Tru64 UNIX                            Little
              6 AIX-Based Systems (64-bit)               Big
              7 Microsoft Windows IA (32-bit)            Little
              8 Microsoft Windows IA (64-bit)            Little
              9 IBM zSeries Based Linux                  Big
             10 Linux IA (32-bit)                        Little
             11 Linux IA (64-bit)                        Little
             12 Microsoft Windows x86 64-bit             Little
             13 Linux x86 64-bit                         Little
             15 HP Open VMS                              Little
             16 Apple Mac OS                             Big
             17 Solaris Operating System (x86)           Little
             18 IBM Power Based Linux                    Big
             19 HP IA Open VMS                           Little
             20 Solaris Operating System (x86-64)        Little
             21 Apple Mac OS (x86-64)                    Little
             22 Linux OS (S64)                           Big
             23 Linux OS (AARCH64)                       Little
    ```
    

## 나의 플랫폼 확인

- 현재 내가 사용하는 플랫폼의 아이디를 확인하기 위해서는 `V$DATABASE` 뷰를 참조한다.
    
    ```sql
    SELECT PLATFORM_ID, PLATFORM_NAME FROM V$DATABASE;
    
    PLATFORM_ID PLATFORM_NAME
    ----------- ----------------------------------------
              6 AIX-Based Systems (64-bit)
    ```
    
    <aside>
    💡
    
    **TTS 를 활용 시, 타겟 플랫폼의 Endian 이 현재 플랫폼의 Endian 과 같을 경우 특별한 Convert 작업이 필요 없으며, 만약 타겟 플랫폼이 현재 플랫폼과 Endian 값이 다를 경우 별도의 Convert 를 수행 해 주어야 한다**
    
    </aside>
    

## TTS 의 일반적인 제약 사항

- 소스와 타겟 데이터베이스는 반드시 서로 호환되는 캐릭터 셋을 가져야 하며 아래의 내용 중 하나는 반드시 맞아야 함
    - 데이터베이스 캐릭터셋은 소스와 타겟 모두 같아야 함
    - 소스 데이터베이스 캐릭터셋이 타겟 데이터베이스의 캐릭터셋에 대한 엄격한(strict) 서브셋이어야 하며, 아래 조건이 일치해야 함
        - 소스 데이터베이스는 10g (10.1.0.3) 또는 이후
        - 전송될 테이블스페이스에 포함된 테이블의 컬럼 중 문자열의 길이를 의미상 길이(nls_length_semantics set to CHAR)로 설정한 컬럼이 없거나, 또는 최대 문자열 길이가 소스 및 타겟 모두 동일 해야 함.
        - 전송될 데이터에 CLOB 데이터타입이 없거나, 또는 소스와 타겟 데이터베이스의 캐릭터 셋이 모두 싱글 바이트이거나 멀티바이터여야 함
    - 소스 데이터베이스가 타겟 데이터베이스 캐릭터셋의 서브셋일 경우, 아래 두 조건이 모두 참이어야 함
        - 소스 데이터베이스는 10g R1 (10.1.0.3) 이전이어야 함
        - 최대 문자열 길이는 소스와 타겟 데이터베이스의 캐릭터셋에서 동일해야 함.
- 소스 및 타겟 데이터베이스는 반드시 호환되는 네셔널 캐릭터셋을 사용하여야 함. 특히 아래 사항이 참이어야 함
    - 네셔널 캐릭터셋은 양 데이터베이스 모두 동일
    - 소스 데이터베이스가 10gR1 (10.1.0.3) 또는 이후의 버전이라면, 전송될 테이블스페이스는 NCHAR, NVARCHAR2, 또는 NCLOB 데이터 타입의 컬럼이 없어야 함.
- non-CDB 환경에서 타겟 데이터베이스로 테이블스페이스를 전송 시 타겟 데이터베이스에 같은 이름의 테이블스페이스가 존재하면 안됨.
- XMLType 을 전송하기 위해서는 아래의 제한 조건이 있음
    - 타겟 데이터베이스는 XML DB 가 설치되어 있어야 함.
    - XMLType 테이블에서 참조하는 스키마들은 XML DB 표준 스키마가 될 수 없음
    - XMLType 테이블을 전송하기 위한 스키마가 타겟에 존재하지 않을 경우 임포트 후 등록됨. 만약 스키마가 타겟에 이미 존재할 경우 임포트간 메시지가 출력됨
    - XMLType 을 포함한 데이터의 메타를 Export / Import 하기 위해서는 반드시 Data Pump 를 사용해야 함.
    - XMLType 을 포함하는 테이블스페이스를 확인하기 위해서는 아래 쿼리 수행
        
        ```sql
        select distinct p.tablespace_name 
          from dba_tablespaces p, dba_xml_tables x, dba_users u, all_all_tables t 
         where t.table_name=x.table_name 
           and t.tablespace_name=p.tablespace_name
            and x.owner=u.username;
        ```
        
- TIMESTAMP WITH LOCAL TIME ZONE(TSLZ) 형식의 데이터가 있는 테이블을 포함한 테이블스페이스를 전송 시 서로간의 다임존이 다를 경우, TSLTZ 데이터를 가진 테이블은 전송되지 않음.
    - Database 의 타임존은 아래 명령을 통해 확인 가능함
        
        ```sql
        SELECT DBTIMEZONE FROM DUAL;
        ```
        
        `ALTER DATABASE` 구문을 이용하여 현 데이터베이스의 타임존을 변경 할 수 있음
        
    - TTS 과정이 종료된 후 TSLTZ 데이터를 가진 테이블들은 별도로 Data Pump 를 통해 데이터를 옮길 수 있음

# Let’s Do It! `XTTS`
## 테스트 서버 정보
**ASIS**

AIX 7.2, 19.29 ASM RAC

COMPATIBLE is 19.0.0

NLS_CHARACTERSET is AL32UTF8

NLS_NCHAR_CHARACTERSET is AL16UTF16

**TOBE**

Linux9, 23.26 Single Filesystem

COMPATIBLE is 19.0.0

NLS_CHARACTERSET is AL32UTF8

NLS_NCHAR_CHARACTERSET is AL16UTF8

## 스키마 구성

### 테이블 스페이스 추가

XTTS 를 수행할 대상 서버에 이관한 대상 테이블 스페이스를 생성한다.

```sql
CREATE TABLESPACE XTTS_TEST_DATA DATAFILE '+DATA' SIZE 10g;

Tablespace created.

CREATE TABLESPACE XTTS_TEST_INDEX DATAFILE '+DATA' SIZE 10g;

Tablespace created.
```

정상적으로 테이블스페이스가 생성 되었는지 확인

```sql
SELECT TABLESPACE_NAME, FILE_NAME, BYTES/1024/1024 AS MB FROM DBA_DATA_FILES WHERE TABLESPACE_NAME LIKE 'XTTS_TEST_%';

TABLESPACE_NAME                FILE_NAME                                                            MB
------------------------------ ------------------------------------------------------------ ----------
XTTS_TEST_DATA                 +DATA/GOODUS/DATAFILE/xtts_test_data.270.1223570029               10240
XTTS_TEST_INDEX                +DATA/GOODUS/DATAFILE/xtts_test_index.269.1223571155              10240
```

### 계정 추가

이관 대상 계정을 생성하고 필요한 권한을 부여한다.

```sql
CREATE USER XTTS_USER IDENTIFIED BY "XTTS_USER" DEFAULT TABLESPACE XTTS_TEST_DATA ACCOUNT UNLOCK;

User Created.
```

해당 테이블스페이스에 `XTTS_USER` 계정이 세그먼트를 생성할 수 있도록 Quota 권한을 부여한다.

```sql
ALTER USER XTTS_USER QUOTA UNLIMITED ON XTTS_TEST_INDEX;
ALTER USER XTTS_USER QUOTA UNLIMITED ON XTTS_TEST_DATA;
```

필요한 권한을 부여한다.

```sql
GRANT CREATE PROCEDURE, CREATE SESSION, CREATE TABLE TO XTTS_USER;
```

### 테스트 데이터 추가

**테이블 생성**

XTTS 를 이용하기 위한 테이블을 생성한다.

```sql
CREATE TABLE XTTS_USER.MIG_DATA(
  TEXTDATA VARCHAR2(30),
  SOMEDATE DATE
) TABLESPACE XTTS_TEST_DATA;
```

**데이터 생성**

```sql
BEGIN
FOR X IN 1..300000 LOOP
 INSERT INTO XTTS_USER.MIG_DATA 
 VALUES(
    DBMS_RANDOM.STRING('U',DBMS_RANDOM.VALUE(1,30)), 
    SYSDATE-(DBMS_RANDOM.VALUE(1,65536))
 );
 IF MOD(X,1000) = 999 THEN
    COMMIT;
 END IF;
END LOOP;
COMMIT;
END;
/
```

데이터 검증

```sql
SELECT COUNT(*) FROM XTTS_USER.MIG_DATA;

  COUNT(*)
----------
    300000
```

**인덱스 생성**

```sql
CREATE INDEX XTTS_USER.IDX_MIG_DATA_01 ON XTTS_USER.MIG_DATA (
  TEXTDATA
) TABLESPACE XTTS_TEST_INDEX
;
```

인덱스 정상 확인

```sql
SET AUTOT TRACE
SELECT * FROM XTTS_USER.MIG_DATA WHERE TEXTDATA = 'VJOIEHPNMJGOFQBEHTD'

Execution Plan
----------------------------------------------------------
Plan hash value: 2960828629

-------------------------------------------------------------------------------------------------------
| Id  | Operation                           | Name            | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                    |                 |     1 |    26 |     3   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| MIG_DATA        |     1 |    26 |     3   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN                  | IDX_MIG_DATA_01 |     1 |       |     1   (0)| 00:00:01 |
-------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("TEXTDATA"='VJOIEHPNMJGOFQBEHTD')

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)

Statistics
----------------------------------------------------------
          0  recursive calls
          0  db block gets
          4  consistent gets
          0  physical reads
          0  redo size
        657  bytes sent via SQL*Net to client
        433  bytes received via SQL*Net from client
          2  SQL*Net roundtrips to/from client
          0  sorts (memory)
          0  sorts (disk)
          1  rows processed
```

**프로시저 생성**

메타 이관시 정상적으로 프로시저 등이 이관 되는지 확인을 위함.

```sql
CREATE OR REPLACE PROCEDURE XTTS_USER.GET_BIRTHDAY(USERNAME IN VARCHAR2) IS
USRNM XTTS_USER.MIG_DATA.TEXTDATA%TYPE;
BIRTHDAY XTTS_USER.MIG_DATA.SOMEDATE%TYPE;
BEGIN
 SELECT SOMEDATE INTO BIRTHDAY
   FROM XTTS_USER.MIG_DATA
  WHERE ROWNUM = 1;
DBMS_OUTPUT.PUT_LINE('Birth Day : '||TO_CHAR(BIRTHDAY,'YYYY/MM/DD'));
END;
/
```

프로시저 테스트

```sql
SET SERVEROUTPUT ON
EXEC XTTS_USER.GET_BIRTHDAY('VJOIEHPNMJGOFQBEHTD');

Birth Day : 1983/07/30

PL/SQL procedure successfully completed.
```

## 이관 준비 [ASIS]

### BCT 설정

최소 다운타임을 위하여, Block Change Tracking 설정을 ASIS Site 에서 수행한다. 이는, RMAN 을 통한 Incremental Backup 을 위해 설정하며, RAC 환경의 경우 공유 디렉토리에 설정한다. 해당 ASIS 서버는 ASM 을 사용하므로 `+RECO` 영역에 생성한다.

```sql
ALTER DATABASE ENABLE BLOCK CHANGE TRACKING USING FILE '+RECO';
```

**Block Tracking File 확인**

`V$BLOCK_CHANGE_TRACKING` 뷰를 확인하여 BCT 설정을 확인할 수 있다.

```sql
SELECT * FROM V$BLOCK_CHANGE_TRACKING

STATUS     FILENAME                                                BYTES     CON_ID
---------- -------------------------------------------------- ---------- ----------
ENABLED    +RECO/GOODUS/CHANGETRACKING/ctf.341.1224173489       11599872          0
```

### Violation Check

TTS 를 수행하기 전, 해당 Tablespace 를 이관할 수 있는지 여부를 `DBMS_TTS` 패키지를 이용하여 확인한다. 해당 패키지로 유효성을 검사한 후 `TRANSPORT_SET_VIOLATIONS` 뷰를 통해 결과를 확인할 수 있다.

```sql
EXEC DBMS_TTS.TRANSPORT_SET_CHECK('XTTS_TEST_INDEX', TRUE);
```

위의 명령은 인덱스 테이블스페이스에 대한 체크를 수행한다. 인덱스는 데이터 테이블이 없으면 존재할 수 없으므로 `TRANSPORT_SET_VIOLATIONS` 를 확인하면 아래와 같은 에러를 확인할 수 있다.

```sql
SELECT *   FROM TRANSPORT_SET_VIOLATIONS;

VIOLATIONS
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
ORA-39907: Index XTTS_USER.IDX_MIG_DATA_01 in tablespace XTTS_TEST_INDEX points to table XTTS_USER.MIG_DATA in tablespace XTTS_TEST_DATA.
```

따라서 Data Tablespace, Index Tablespace 를 같이 확인한다.

```sql
DECLARE
  TSLIST   VARCHAR2(4000);
BEGIN
  SELECT LISTAGG(TABLESPACE_NAME,',') WITHIN GROUP (ORDER BY TABLESPACE_NAME)
    INTO TSLIST
    FROM DBA_TABLESPACES WHERE TABLESPACE_NAME LIKE 'XTTS%';
  DBMS_TTS.TRANSPORT_SET_CHECK(
     TSLIST,
     TRUE,
     TRUE
  );
END;
/

PL/SQL procedure successfully completed.

SELECT * FROM TRANSPORT_SET_VIOLATIONS;

no rows selected
```

### Schema 생성 스크립트 준비

TOBE  에 미리 생성할 스미카를 미리 확인하여 준비한다. 여기서는 `XTTS_USER` 가 그 대상이 된다.

```sql
SET LONG 99999
SET PAGES 500000
SELECT DBMS_METADATA.GET_DDL('USER','XTTS_USER') FROM DUAL;

DBMS_METADATA.GET_DDL('USER','XTTS_USER')
--------------------------------------------------------------------------------

   CREATE USER "XTTS_USER" IDENTIFIED BY VALUES 'S:A381B827
56754444B2CC8AA9FE6BF6F78756FD6A14C2CD86
53A207F10D7E;T:D9D5D93BC278E997A6BB06BDB
5BE732E5361D8997D86006CD87A6D1313B6B2F8A
CA3804197AC118C92B2791F1870A9FF560AC7F07
C78006A3C2ED0798A45E9F8A4AD328B285CA9163
525997DADB965CC'
      DEFAULT TABLESPACE "XTTS_TEST_DATA"
      TEMPORARY TABLESPACE "TEMP";
```

## 이관준비 [TOBE]

### Schema 생성

미리 스키마를 생성한다. 단, 기본 테이블스페이스는 다른 테이블스페이스로 지정하여 생성한다.

```sql
CREATE USER "XTTS_USER" IDENTIFIED BY VALUES 'S:A381B82756754444B2CC8AA9FE6BF6F78756FD6A14C2CD8653A207F10D7E;T:D9D5D93BC278E997A6BB06BDB5BE732E5361D8997D86006CD87A6D1313B6B2F8ACA3804197AC118C92B2791F1870A9FF560AC7F07C78006A3C2ED0798A45E9F8A4AD328B285CA9163525997DADB965CC'
DEFAULT TABLESPACE "USERS"
TEMPORARY TABLESPACE "TEMP";

User Created.
```

### Platform 확인

Convert 를 수행 하여야 하므로, 미리 현재의 플랫폼을 확인한다.

```sql
SELECT PLATFORM_NAME FROM V$DATABASE;

PLATFORM_NAME
--------------------------------------------------------------------------------
Linux x86 64-bit
```

## Backup 수행 | ASIS

### Level 0 Backup 수행

이제 준비 및 확인은 끝났으니 ASIS 에서 해당 테이블스페이스를 백업 받는다.

*target Dir : /u01/oracle/imsi*

```sql
RUN {
 ALLOCATE CHANNEL D1 TYPE DISK;
 BACKUP AS COPY INCREMENTAL LEVEL 0 TAG 'XTTS_INC_0' TABLESPACE XTTS_TEST_DATA FORMAT '/u01/oracle/imsi/backup_level0_%T_%U.bk';
 BACKUP AS COPY INCREMENTAL LEVEL 0 TAG 'XTTS_INC_0' TABLESPACE XTTS_TEST_INDEX FORMAT '/u01/oracle/imsi/backup_level0_%T_%U.bk';
}
```

아래와 같이 백어비 정상적으로 받아졌음을 확인할 수 있다.

```bash
ls -lrt /u01/oracle/imsi/

total 41943080
-rw-r-----    1 oracle   dba      10737426432 Feb  2 17:20 backup_level0_20260202_data_D-GOODUS_I-3837308135_TS-XTTS_TEST_DATA_FNO-6_054fetu2.bk
-rw-r-----    1 oracle   dba      10737426432 Feb  2 17:21 backup_level0_20260202_data_D-GOODUS_I-3837308135_TS-XTTS_TEST_INDEX_FNO-7_064fetvp.bk
```

### 백업 파일 전송

받은 백업 파일을 TOBE 측으로 전송한다.

```bash
scp * oracle@<Linux Server IP>:~/backup/
```

### Convert

이제 TOBE 서버에서 엔디안 포멧에 맞게 풀백업본을 Convert 수행한다. Convert 는 RMAN 으로 수행한다.

- ASIS Platform : AIX-Based Systems (64-bit)

```sql
run
{
convert from platform 'AIX-Based Systems (64-bit)' datafile '/home/oracle/backup/backup_level0_20260202_data_D-GOODUS_I-3837308135_TS-XTTS_TEST_DATA_FNO-6_054fetu2.bk' format '+DATA';
convert from platform 'AIX-Based Systems (64-bit)' datafile '/home/oracle/backup/backup_level0_20260202_data_D-GOODUS_I-3837308135_TS-XTTS_TEST_INDEX_FNO-7_064fetvp.bk' format '+DATA';
} 
```

- Command Result
    
    ```sql
    Starting conversion at target at 03-FEB-26
    using target database control file instead of recovery catalog
    allocated channel: ORA_DISK_1
    channel ORA_DISK_1: SID=152 instance=ORCLDB1 device type=DISK
    channel ORA_DISK_1: starting datafile conversion
    input file name=/home/oracle/backup/backup_level0_20260202_data_D-GOODUS_I-3837308135_TS-XTTS_TEST_DATA_FNO-6_054fetu2.bk
    converted datafile=+**DATA/ORCLDB/DATAFILE/xtts_test_data.270.1224335777**
    channel ORA_DISK_1: datafile conversion complete, elapsed time: 00:00:45
    Finished conversion at target at 03-FEB-26
    
    Starting conversion at target at 03-FEB-26
    using channel ORA_DISK_1
    channel ORA_DISK_1: starting datafile conversion
    input file name=/home/oracle/backup/backup_level0_20260202_data_D-GOODUS_I-3837308135_TS-XTTS_TEST_INDEX_FNO-7_064fetvp.bk
    converted datafile=**+DATA/ORCLDB/DATAFILE/xtts_test_index.269.1224335823**
    channel ORA_DISK_1: datafile conversion complete, elapsed time: 00:00:45
    Finished conversion at target at 03-FEB-26
    ```
    
    정상적으로 각각 `+DATA` Diskgroup 에 데이터파일이 생성된 것을 확인할 수 있다.
    

## 증분 백업 수행 | ASIS

### 추가 데이터 생성

XTTS 의 단점으로는 일단 초기 복원한 데이터파일의 내용을 확인할 수 없다는 것이다. 정상적으로 Convert 여부는 RMAN 의 로그 출력을 통해 확인하는 수 밖에 없다. 최소한의 다운타임이 목표이기 때문에 이제 ASIS 의 추가적인 증분 데이터를 입력한다. 기존 10만건에서 추가적으로 10만건을 생성한다.

```sql
BEGIN
FOR X IN 1..100000 LOOP
 INSERT INTO XTTS_USER.MIG_DATA 
 VALUES(
    DBMS_RANDOM.STRING('U',DBMS_RANDOM.VALUE(1,30)), 
    SYSDATE-(DBMS_RANDOM.VALUE(1,65536))
 );
 IF MOD(X,1000) = 999 THEN
    COMMIT;
 END IF;
END LOOP;
COMMIT;
END;
/
```

```sql
 SELECT COUNT(*) FROM XTTS_USER.MIG_DATA;

  COUNT(*)
----------
    400000
```

### 증분 백업 수행

ASIS 에서 증분 백업을 수행한다.

```sql
RUN {
 ALLOCATE CHANNEL D1 TYPE DISK;
 BACKUP INCREMENTAL LEVEL 1 TAG 'XTTS_INC_1' TABLESPACE XTTS_TEST_DATA, XTTS_TEST_INDEX FORMAT '/u01/oracle/imsi/backup_level1_%T_%U.bk';
}
```

- Log
    
    ```sql
    using target database control file instead of recovery catalog
    allocated channel: D1
    channel D1: SID=4237 instance=GOODUS1 device type=DISK
    
    Starting backup at 04-FEB-26
    channel D1: starting incremental level 1 datafile backup set
    channel D1: specifying datafile(s) in backup set
    input datafile file number=00006 name=+DATA/GOODUS/DATAFILE/xtts_test_data.270.1224335777
    input datafile file number=00007 name=+DATA/GOODUS/DATAFILE/xtts_test_index.269.1224335823
    channel D1: starting piece 1 at 04-FEB-26
    channel D1: finished piece 1 at 04-FEB-26
    piece handle=/u01/oracle/imsi/bakcup_level1_20260204_0e4fjmv5_14_1_1.bk tag=XTTS_INC_1 comment=NONE
    channel D1: backup set complete, elapsed time: 00:00:01
    Finished backup at 04-FEB-26
    
    Starting Control File and SPFILE Autobackup at 04-FEB-26
    piece handle=+RECO/GOODUS/AUTOBACKUP/2026_02_04/s_1224334311.354.1224334311 comment=NONE
    Finished Control File and SPFILE Autobackup at 04-FEB-26
    
    released channel: D1
    ```
    

> Incremental Backup 시에는 `BACKUP AS COPY` 로 하지 않는다.
> 

증분 백업을 수행한 백업 파일을 다시 TOBE 로 옮긴다.

## 증분백업 적용 | TOBE

### 증분백업 Convert

일반 이미지 파일이 아닌 Backup piece  는 Convert 를 수행할 수 없다.

```sql
set serveroutput on;
set termout on;
set verify off;
DECLARE
    handle varchar2(512) ;
    comment varchar2(80) ;
    media varchar2(80) ;
    concur boolean ;
    recid number ;
    stamp number ;
    platfrmto number;
    same_endian number := 1;
    devtype VARCHAR2(512);
BEGIN
    BEGIN
        sys.dbms_backup_restore.restoreCancel(TRUE);
        devtype := sys.dbms_backup_restore.deviceAllocate;
        sys.dbms_backup_restore.backupBackupPiece(
            bpname => '/home/oracle/backup/bakcup_level1_20260204_0e4fjmv5_14_1_1.bk',
            fname => '/home/oracle/backup/bakcup_level1_20260204_0e4fjmv5_14_1_1.bk.conv',
            handle => handle, media => media, comment => comment,
            concur => concur, recid => recid, stamp => stamp, check_logical => FALSE,
            copyno => 1, deffmt => 0, copy_recid => 0, copy_stamp => 0,
            npieces => 1, dest => 0,
            pltfrmfr => 6);
    EXCEPTION
        WHEN OTHERS
        THEN
            dbms_output.put_line ('ERROR IN CONVERSION ' || SQLERRM);
    END ;
    sys.dbms_backup_restore.deviceDeallocate;
    dbms_output.put_line('Handle : '||handle);
END;
/
 
Handle : **/home/oracle/backup/bakcup_level1_20260204_0e4fjmv5_14_1_1.bk.conv**

PL/SQL procedure successfully completed.
```

컨버트가 완료되면 해당 디렉토리에 `/home/oracle/backup/bakcup_level1_20260204_0e4fjmv5_14_1_1.bk.conv` ****가 생성된다. 아래 인풋 파라메터만 각자의 상황에 맞게 변경하여 준다.

**bpname** : Backup Piece Name

ASIS 에 받아온 백업 피스의 이름

**fname** : File name

Convert 후 생성할 파일 명

**pltfrmfr** : Platform From

해당 백업 피스를 받은 ASIS 시스템의 플랫폼 번호.

### 증분백업 적용

Convert 에 성공한 백업 파일을 기존의 파일에 적용한다.

```sql
set serveroutput on
DECLARE
    d varchar2(512);
    h varchar2(512);
    t varchar2(30);
    b1 boolean;
    b2 boolean;
    DONE boolean;
    FAILOVER boolean;
BEGIN
    d := sys.dbms_backup_restore.deviceAllocate;
    dbms_output.put_line(d);
    sys.dbms_backup_restore.applysetdatafile(
      check_logical=>false, 
      cleanup=>false);
    sys.dbms_backup_restore.applyDatafileTo(
      dfnumber=>6 , 
      toname => '+DATA/GOODUS/DATAFILE/xtts_test_data.270.1224335777',
      fuzziness_hint=>0,
      max_corrupt =>0,
      islevel0=>0,
      recid=>0,
      stamp=>0);
    sys.dbms_backup_restore.applyDatafileTo(
      dfnumber=>7 , 
      toname => '+DATA/GOODUS/DATAFILE/xtts_test_index.269.1224335823',
      fuzziness_hint=>0,
      max_corrupt =>0,
      islevel0=>0,
      recid=>0,
      stamp=>0);
    sys.dbms_backup_restore.restoreSetPiece(
      handle=>'/home/oracle/backup/bakcup_level1_20260204_0e4fjmv5_14_1_1.bk.conv',
      tag=>null,
      fromdisk=>true,
      recid=>0,
      stamp=>0);
    sys.dbms_backup_restore.restoreBackupPiece(
      done=>DONE, 
      params=>null, 
      outhandle=>h, 
      outtag=>t, 
      FAILOVER=>FAILOVER);
    sys.dbms_backup_restore.restoreCancel(TRUE);
    sys.dbms_backup_restore.deviceDeallocate;
END;
/
```

## 최종 증분 백업

### 데이터 셋 추가 | ASIS

마지막으로 최종 증분 백업을 하기 위해서 한번 더 데이터셋을 증가 시켜준다.

```sql
BEGIN
FOR X IN 1..100000 LOOP
 INSERT INTO XTTS_USER.MIG_DATA 
 VALUES(
    DBMS_RANDOM.STRING('U',DBMS_RANDOM.VALUE(1,30)), 
    SYSDATE-(DBMS_RANDOM.VALUE(1,65536))
 );
 IF MOD(X,1000) = 999 THEN
    COMMIT;
 END IF;
END LOOP;
COMMIT;
END;
/

PL/SQL procedure successfully completed.

SQL> select count(*) from xtts_user.mig_data;

  COUNT(*)
----------
    50000
```

### 최종 증분분 백업 | ASIS

**Tablespace Readonly**

최종 증분분을 백업하기 위해서 대상 테이블 스페이스를 READ ONLY 로 변경 해 주어야 한다.

```sql
ALTER TABLESPACE XTTS_TEST_DATA READ ONLY;
ALTER TABLESPACE XTTS_TEST_INDEX READ ONLY;
```

정상적으로 Readonly 모드로 변경 되었는지 확인한다.

```sql
SELECT TABLESPACE_NAME, STATUS 
  FROM DBA_TABLESPACES 
 WHERE TABLESPACE_NAME LIKE 'XTTS%'

TABLESPACE_NAME                STATUS
------------------------------ ---------
XTTS_TEST_DATA                 **READ ONLY**
XTTS_TEST_INDEX                **READ ONLY**
```

**증분 백업 수행**

최종적인 증분 백업을 수행한다.

```sql
BACKUP INCREMENTAL LEVEL 1 TAG 'XTTS_INC_1' TABLESPACE XTTS_TEST_DATA, XTTS_TEST_INDEX FORMAT '/u01/oracle/imsi/backup_level1_%T_%U.bk';
```

- log
    
    ```sql
    BACKUP INCREMENTAL LEVEL 1 TAG 'XTTS_INC_1' TABLESPACE XTTS_TEST_DATA, XTTS_TEST_INDEX FORMAT '/u01/oracle/imsi/backup_level1_%T_%U.bk';
    Starting backup at 04-FEB-26
    using target database control file instead of recovery catalog
    allocated channel: ORA_DISK_1
    channel ORA_DISK_1: SID=5205 instance=GOODUS1 device type=DISK
    channel ORA_DISK_1: starting incremental level 1 datafile backup set
    channel ORA_DISK_1: specifying datafile(s) in backup set
    input datafile file number=00006 name=+DATA/GOODUS/DATAFILE/xtts_test_data.270.1223570029
    input datafile file number=00007 name=+DATA/GOODUS/DATAFILE/xtts_test_index.269.1223571155
    channel ORA_DISK_1: starting piece 1 at 04-FEB-26
    channel ORA_DISK_1: finished piece 1 at 04-FEB-26
    piece handle=**/u01/oracle/imsi/backup_level1_20260204_0g4fjvla_16_1_1.bk** tag=XTTS_INC_1 comment=NONE
    channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
    Finished backup at 04-FEB-26
    
    Starting Control File and SPFILE Autobackup at 04-FEB-26
    piece handle=+RECO/GOODUS/AUTOBACKUP/2026_02_04/s_1224343211.357.1224343211 comment=NONE
    Finished Control File and SPFILE Autobackup at 04-FEB-26
    ```
    

`/u01/oracle/imsi/backup_level1_20260204_0g4fjvla_16_1_1.bk` 파일명으로 백업 피스가 생성되었으며, 이 백업을 다시 TOBE 로 전송한다.

### 최종 증분분 컨버트 | TOBE

이미 수행했던 동일한 방법을 통해 증분분을 Convert 수행한다.

```sql
set serveroutput on;
set termout on;
set verify off;
DECLARE
    handle varchar2(512) ;
    comment varchar2(80) ;
    media varchar2(80) ;
    concur boolean ;
    recid number ;
    stamp number ;
    platfrmto number;
    same_endian number := 1;
    devtype VARCHAR2(512);
BEGIN
    BEGIN
        sys.dbms_backup_restore.restoreCancel(TRUE);
        devtype := sys.dbms_backup_restore.deviceAllocate;
        sys.dbms_backup_restore.backupBackupPiece(
            bpname => '/home/oracle/backup/**backup_level1_20260204_0g4fjvla_16_1_1.bk**',
            fname => '/home/oracle/backup/**backup_level1_20260204_0g4fjvla_16_1_1.bk.conv**',
            handle => handle, media => media, comment => comment,
            concur => concur, recid => recid, stamp => stamp, check_logical => FALSE,
            copyno => 1, deffmt => 0, copy_recid => 0, copy_stamp => 0,
            npieces => 1, dest => 0,
            pltfrmfr => 6);
    EXCEPTION
        WHEN OTHERS
        THEN
            dbms_output.put_line ('ERROR IN CONVERSION ' || SQLERRM);
    END ;
    sys.dbms_backup_restore.deviceDeallocate;
    dbms_output.put_line('Handle : '||handle);
END;
/

Handle : /home/oracle/backup/backup_level1_20260204_0g4fjvla_16_1_1.bk.conv

PL/SQL procedure successfully completed.
```

### 최종 증분분 적용 | TOBE

```sql
set serveroutput on
DECLARE
    d varchar2(512);
    h varchar2(512);
    t varchar2(30);
    b1 boolean;
    b2 boolean;
    DONE boolean;
    FAILOVER boolean;
BEGIN
    d := sys.dbms_backup_restore.deviceAllocate;
    dbms_output.put_line(d);
    sys.dbms_backup_restore.applysetdatafile(
      check_logical=>false, 
      cleanup=>false);
    sys.dbms_backup_restore.applyDatafileTo(
      dfnumber=>6 , 
      toname => '+DATA/GOODUS/DATAFILE/xtts_test_data.270.1224335777',
      fuzziness_hint=>0,
      max_corrupt =>0,
      islevel0=>0,
      recid=>0,
      stamp=>0);
    sys.dbms_backup_restore.applyDatafileTo(
      dfnumber=>7 , 
      toname => '+DATA/GOODUS/DATAFILE/xtts_test_index.269.1224335823',
      fuzziness_hint=>0,
      max_corrupt =>0,
      islevel0=>0,
      recid=>0,
      stamp=>0);
    sys.dbms_backup_restore.restoreSetPiece(
      handle=>'/home/oracle/backup/backup_level1_20260204_0g4fjvla_16_1_1.bk.conv',
      tag=>null,
      fromdisk=>true,
      recid=>0,
      stamp=>0);
    sys.dbms_backup_restore.restoreBackupPiece(
      done=>DONE, 
      params=>null, 
      outhandle=>h, 
      outtag=>t, 
      FAILOVER=>FAILOVER);
    sys.dbms_backup_restore.restoreCancel(TRUE);
    sys.dbms_backup_restore.deviceDeallocate;
END;
/
```

## ASIS Meta Export

### Tablespace metadata export

테이블스페이스를 익스포트 하기 위한 디렉토리를 생성한다.

```sql
create directory xtts_test as '/u01/oracle/imsi';
```

생성된 디렉토리로 덤프파일을 내린다.

```sql
expdp dumpfile=xtts_test.dmp logfile=xtts_test.log directory=xtts_test transport_tablespaces=xtts_test_data,xtts_test_index
```

Export 가 끝나면 TTS 대상에 대한 정보가 출력되니 참고하면 된다.

```sql
Datafiles required for transportable tablespace XTTS_TEST_DATA:
  **+DATA/GOODUS/DATAFILE/xtts_test_data.270.1223570029**
Datafiles required for transportable tablespace XTTS_TEST_INDEX:
  **+DATA/GOODUS/DATAFILE/xtts_test_index.269.1223571155**
Job "SYS"."SYS_EXPORT_TRANSPORTABLE_01" successfully completed at Wed Feb 4 15:39:38 2026 elapsed 0 00:01:02
```

생성된 덤프라일을 TOBE 쪽으로 전송한다.

## TOBE Meta Import

### Metadata Import

**디렉토리 생성**

TOBE 에서 덤프파일을 받은 OS 디렉토리에 대하여 데이터베이스 디렉토리 객체를 생성한다.

```sql
create directory xtts_test as '/home/oracle/backup';
```

메타데이터를 Import 수행한다. 데이터파일이 많을 경우 parfile 을 활용하는것도 좋다.

```sql
 impdp dumpfile=xtts_test.dmp logfile=xtts_test_import.log \
 transport_datafiles='+DATA/GOODUS/DATAFILE/xtts_test_data.270.1224335777', \
 '+DATA/GOODUS/DATAFILE/xtts_test_index.269.1224335823' \
 directory=xtts_test
```

메타가 임포트가 완료되었다면 정상적으로 테이블스페이스가 생성 되었는지 확인한다.

```sql
SELECT TABLESPACE_NAME, STATUS FROM DBA_TABLESPACES;

TABLESPACE_NAME                STATUS
------------------------------ ---------
..
XTTS_TEST_DATA                 READ ONLY
XTTS_TEST_INDEX                READ ONLY
```

### 데이터 검증

Tablespace 가 정상적으로 Transport 되었다면 현재 상태는 READ ONLY 이다. 이 상태에서 데이터 검증을 수행한다.

```sql
SELECT COUNT(*) FROM XTTS_USER.MIG_DATA;

  COUNT(*)
----------
    500000
```

정상적으로 데이터가 마이그레이션 된 것을 확인할 수 있다. 마지막으로 해당 테이블스페이스를 Read-Write 모드로 변경 해 줌으로서 마무리를 지을 수 있다.

```sql
ALTER TABLESPACE XTTS_TEST_DATA READ WRITE;
ALTER TABLESPACE XTTS_TEST_INDEX READ WRITE;
```

### 메타 검증

TOBE 에서 오브젝트를 확인 해 보면 위에서 테스트로 생성한 Procedure 는 이관되지 않은 것을 확인할 수 있다.

```sql
SELECT OWNER, OBJECT_NAME, OBJECT_TYPE FROM DBA_OBJECTS WHERE OWNER = 'XTTS_USER'

OWNER           OBJECT_NAME                    OBJECT_TYPE
--------------- ------------------------------ -----------------------
XTTS_USER       MIG_DATA                       TABLE
XTTS_USER       IDX_MIG_DATA_01                INDEX
```

따라서 세그먼트를 제외한 나머지 오브젝트들은, 별도로 메타 Export 를 통해 수행한다. 이 때 TOBE 에 Import 시 `TABLE_EXISTS_ACTION` 파라메터는 반드시 `SKIP` 으로 둔다. (기본값은 SKIP 이나, 혹시 모르니 명시적으로 SKIP 구문을 넣을 것을 권고합니다.)

# FailCase

- 데이터 파일은 Image Backup 으로 받되, Incremental Level 1 백업은 백업피스로 받는다.
- 지속적인 Incremental Backup 을 적용 중, 해당 테이블 스페이스에 데이터파일이 추가 될 경우, Incremental Backup 을 받고 추가적으로 신규 추가된 데이터파일에 대하여 Level 0 의 이미지 백업을 받는다.
    - TOBE 에 Incremental Backup 을 적용 후 추가된 데이터파일을 TOBE 에서 Convert 를 수행한다.
    - 이후 나오는 Incremental 백업을 적용 시 신규 추가한 데이터파일도 같이 적용한다.
