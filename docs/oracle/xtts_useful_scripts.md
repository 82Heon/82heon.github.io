---
title: XTTS Useful Scripts (AIX to LINUX)
parent: Oracle
nav_order: 1
---

{:toc}

# 개요

## 개발 동기

xTTS 를 사용하기 위해 기존의 문서들을 확인하면, 일반적으로 테스트의 작은 테이블들을 통해 테스트한 내용밖에 찾아볼 수 없다. 그러나, 실제 U2L 프로젝트를 진행 시 데이터파일이 상당히 많을 경우, (이번 경험의 경우, 데이터파일 3000개 이상) xTTS 를 사용하기 위한 준비가 만만치 않다. 특히 xTTS 를 사용하면서, Level 1 백업을 받을 시, 할당된 채널별로 각 피스가 포함하는 데이터파일의 정보를 알아야 하며, 이 정보를 이용해 TOBE 에 이미 컨버트 된 데이터파일의 파일명을 알아야 Level 1 백업피스를 적용할 수 있는데, 이걸 일일이 비교하는 건 사람이 하기에는 특히 불가능하다. 특히 ASM 을 사용할 경우 OMF 에 의해 기존과 파일명이 달라지기 때문에 더욱 자동화 된 스크립트가 필요했다.

이번 문서에서는 xTTS 를 활용하여 U2L 프로젝트를 진행 시 사용할 수 있는 유용한 스크립트를 기록해 놓는다.

> 이 스크립트는 **AIX to Linux** 시 사용할 수 있는 스크립트이다.
> 

아래 환경에서의 로그는 Asis 와 Tobe 간 백업의 경로가 /backup 이라는 NAS 를 통해 이루어지는 것을 가정한다.

## 원리

이 스크립트들은 RMAN에서 각각 Level 0 및  Level 1 백업을 수행시 발생하는 **각 로그를 기반**으로 사용자가 편하게 수행할 수 있는 스크립트를 생성한다. 따라서 각 RMAN을 수행할 시 아래와 같이 `log` 옵션을 통해 rman 로그를 특정 위치에 저장하여야 한다.

```bash
rman target / log=...
```

## 순서 요약

| 스크립트 | 대상 | 사용 | 결과예시 | 비고 |
| --- | --- | --- | --- | --- |
| 01.extract_lvl0.ksh | ASIS(AIX) | $0 (lvl0_rman_log) (output) | lv0.log | Level0 백업 로그 읽어 파싱 |
| conv.run | TOBE(LINUX) |  | conv_0_run.run | 리눅스로 컨버트 하는 RMAN Script 생성  |
| 02.convert_log_parse.bash | TOBE(LINUX) | $0 (convlog) (outupt) | lv0_conv.log | 컨버트 시 생성된 로그를 파싱 |
| conv_l1.sh | TOBE(LINUX) | $0 (lvl1_rman_log) (output) | conv.sql | 레벨1 백업피스를 변환하는 스크립트 생성 |
| 03.incre_file.ksh | ASIS(AIX) | $0 (lvl1_rman_log) (output) | lv1.log | lv1 백업에 대한 로그를 파싱 (핸들별 파일구성 확인) |
| 04.log_merge.bash | TOBE(LINUX) | $0 lv0.log lv0_conv.log lv1.log (output) | merged_log | 파싱한 로그 병합 |
| 05.incre_apply_gen.bash | TOBE(LINUX) | $0 merged_log (output) | recover.sql | Level 1 백업 적용 스크립트 생성 |
| 06.make_par.bash | TOBE(LINUX) | $0 (lvl0_rman_log) (output) | xtts_imp_par.par | xtts import par 파일 생성 |

# xTTS 절차에 따른 스크립트

## (AIX) Level 0 Backup Log Parsing : `01.extract_lvl0.ksh`

### **Script**

```bash
#!/bin/ksh

if [ $# -ne 2 ]; then
  echo "Usage: $0 <rman level 0 incre backup log file> <output_filename>"
  echo $#
  exit 1
fi

awk '
/starting datafile copy/ {
 work_channel=$2;
}

/input datafile file number=/ {
 match($0, /number=[0-9]+/);
 fno[work_channel]=substr($0,RSTART+7,RLENGTH-7)+0;
 match($0, "/DATAFILE/[+A-Za-z0-9._-]+");
 srcfile[work_channel]=substr($0,RSTART+9,RLENGTH-9);
}

/output file name=/ {
 match($0,/name=\/ora_mig\/[a-zA-Z0-9\/._+]+/);
 targetfile=substr($0,RSTART+5,RLENGTH-5);
}

/datafile copy complete/ {
 work_channel=$2;
 print fno[work_channel],",",srcfile[work_channel],",",targetfile;
 delete srcfile[work_channel];
 delete fno[work_channel];
}' $1 > $2
```

### **Syntax**

```bash
./01.extract_lvl0.ksh <Rman Level 0 Backup Log File> <Output file>
```

### **Usage**

```bash
./01.extract_lvl0.ksh /backup/level0_backup_rman.log /backup/level0_log_parsing.log
```

### **Description**

최초 받은 Level 0 RMAN 로그의 파일을 파싱한다. xTTS 를 위한 Level 0 백업은 Image Copy 방식의 백업이다. 이 로그를 통해 해당 스크립트를 수행하면 결과적으로 **파일번호, 기존 파일명, 백업 파일명** 을 CVS 형식으로 기록한다. 이 스크립트는 ASIS 인 AIX 서버에서 실행한다.

## (LINUX) Convert Generator : `conv.run`

```bash
ls -İrt **<image_backup_location>** | awk '{print $NF}' | sed -nE '/**[a-zA-20-9]+\.[O-9]+\.[0-9]+**/p' | awk '
BEGIN {
 printf "run {\n";
 printf " allocate channel ch01 device type disk;\n";
 printf " allocate channel ch02 device type disk;\n";
 printf " allocate channel ch03 device type disk;\n";
 printf " allocate channel ch04 device type disk;\n";
 printf " allocate channel ch05 device type disk;\n";
 printf " allocate channel ch06 device type disk;\n";
 printf " allocate channel ch07 device type disk;\n";
 printf " allocate channel ch08 device type disk;\n";
 printf " allocate channel ch09 device type disk;\n";
 printf " allocate channel ch10 device type disk;\n";
 printf " allocate channel ch11 device type disk;\n";
 printf " allocate channel ch12 device type disk;\n";
 printf " convert from platform = \047**AIX-Based Systems (64-bit)**\047 DATAFILE \n";
 i=1;
}

{
 if (i == 1) {
  printf "  \047/**<image_backup_location>**/%s\047\n", $NF;
  i++
 } else {
  printf "  ,\047/**<image_backup_location>**/%s\047\n", $NF;
 }
}

END {
 printf " FORMAT \047**<+ASM_DISKGROUP>**\047;\n";
 printf " release channel ch01;\n";
 printf " release channel ch02;\n";
 printf " release channel ch03;\n";
 printf " release channel ch04;\n";
 printf " release channel ch05;\n";
 printf " release channel ch06;\n";
 printf " release channel ch07;\n";
 printf " release channel ch08;\n";
 printf " release channel ch09;\n";
 printf " release channel ch10;\n";
 printf " release channel ch11;\n";
 printf " release channel ch12;\n";
 printf "}\n";
}' > conv_0_run.run

printf " run rman script using .. \n";
printf "   nohup rman target / log=<log_location>/logfile cmdfile=conv_0_run.run &\n"
```

### **Description**

해당 스크립트를 자신의 상황과 맞춰서 일부 수정한다. 붉은 색 볼드체로 된 부분 및 푸른색 볼드체를 자신의 환경에 맞게 수정하고 수행하면 `conv_0_run.run` 파일명의 RMAN 스크립트가 생성된다.

- <image_backup_location>
    
    최초 TTS 를 수행하기 위한 Image Backup (Level 0) 파일의 위치이다.
    
- <+ASM_DISKGROUP>
    
    백업 받은 이미지 파일을 컨버트 후 저장할 디스크 그룹 및 위치를 지정한다.
    

생성된 스크립트를 RMAN 을 통해 수행한다. (TOBE 측)

```bash
nohup rman target / log=<convert log file> cmdfile=conv_0_run.run &
```

## (Linux) Level 0 Convert Log Parsing : `02.convert_log_parse.bash`

### **Script**

```bash
#!/bin/bash

if [ $# -ne 2 ]; then
  echo "Usage: $0 <rman convert log file> <output_filename>"
  exit 1
fi

INPUTFILE=$1
OUTPUTFILE=$2

awk '
/starting datafile conversion/ {
 work_channel = $2;
}

/input file name=/ {
 match($0, /input file name=[\/a-zA-Z0-9._-]+/);
 srcfile[work_channel]=substr($0,RSTART+16);
 conv_start_txt[work_channel] = $0
}

/converted datafile=/ {
 match($0, /datafile=[+\/a-zA-Z_.0-9-]+/);
 targetfile=substr($0,RSTART+9, RLENGTH-9);
 conv_end_txt = $0;
}

/datafile conversion complete/ {
 work_channel = $2;
 print srcfile[work_channel],",",targetfile;
 delete srcfile[work_channel];
}' $INPUTFILE > $OUTPUTFILE
```

### Syntax

```bash
./02.convert_log_parse.bash <Datafile Convert log> <output_filename>
```

### Usage

```bash
./02.convert_log_parse.bash rman_convert_log.log rman_convert_log_parsing.log
```

### Description

RMAN 을 통해 ASIS 에서 백업 받은 데이터파일을 Linux 용으로 컨버트를 수행할 시 생성되는 로그를 활용한다. 결과적으로 해당 로그를 통해 **백업파일, 컨버트된 파일** 의 정보를 CSV 형식으로 저장한다.

## (AIX) Level 1 Incre Backup Log Parsing : `03.incre_file.ksh`

### Script

```bash
#!/bin/ksh

if [ $# -ne 2 ]; then
  echo "Usage: $0 <rman level 1 incre backup log file> <output_filename>"
  exit 1
fi

awk '
/specifying datafile/ {
 work_channel = $2;
}

/input datafile file number/ {
 match($0, /number=[0-9]+/)
 fno=substr($0,RSTART+7,RLENGTH-7)+0;
 match($0, "name=[/A-Za-z0-9_.+-]*");
 filename=substr($0,RSTART+5);
 included_files[work_channel,fno]=filename;
}

/finished piece/ {
 work_channel=$2;
}

/piece handle/ {
 match($0,/piece handle=\/[A-Za-z0-9\/_-]*/);
 handle_name=substr($0, RSTART+13, RLENGTH-13);
 for (f in included_files) {
  split(f, indices, SUBSEP);
  if (indices[1] == work_channel) {
   print handle_name,",",indices[2]+0,",",included_files[f];
   delete included_files[f]
  }
 }
}

/Starting Control File/ {
 exit 0
} ' $1 > $2

```

### Syntax

```bash
./03.incre_file.ksh <Level1 incre backup log> <Output file>
```

### Usage

```bash
./03.incre_file.ksh /backup/rman_inc1_backup.log /backup/rman_inc1_log_parsing.log
```

### Description

레벨 1 증분  백업을 수행한 RMAN 로그를 통해 각 `핸들명과 해당 핸들에 포함된 데이터파일 목록`을 CSV 형태로 저장한다.

아마 이 스크립트를 수행한 이후 Level 1 백업에 대하여 Convert 를 수행할 것이다. 앞으로의 편의를 위해 Convert 된 파일명은 기존의 Level1 백업 파일명에 `_conv` 만 붙여 핸들을 컨버트 한다. 이는 나중에 로그를 모두 병합하고, 해당 로그를 통해 복구 스크립트를 생성할 시 사용자의 개입을 가급적 하지 않기 위함이다.

## (Linux) Log Merge : `04.log_merge.bash`

### Script

```bash
#!/bin/bash

if [ $# -ne 4 ]; then
  echo "Usage: $0 <lvl 0 rman backup log> <convert rman log> <incre1 rman log> <output_filename>"
  exit 1
fi

FILE_A=$1
FILE_B=$2
FILE_C=$3
OUTPUTFILE=$4

for f in $FILE_A $FILE_B $FILE_C; do
        if [ ! -f "$f" ]; then
                echo "Error: File '$f' not found."
                exit 1;
        fi
done

awk -F ',' -v outfile="$OUTPUTFILE" '
ARGIND==1 {
        gsub(/^[ \t]+|[ \t]+$/, "", $1);
        gsub(/^[ \t]+|[ \t]+$/, "", $3);
        a[$3] = $1;
        next;
}

ARGIND==2 {
        gsub(/^[ \t]+|[ \t]+$/, "", $1);
        gsub(/^[ \t]+|[ \t]+$/, "", $2);
        if ($1 in a) {
                b[$1] = $2;
        }
        next;
}

ARGIND==3 {
        gsub(/^[ \t]+|[ \t]+$/, "", $1)
        gsub(/^[ \t]+|[ \t]+$/, "", $2)
        gsub(/^[ \t]+|[ \t]+$/, "", $3)

        for (key in a){
                if (a[key] == $2) {
                        result[NR]=sprintf("%s, %s, %s", $1, a[key], b[key]);
                }
        }
}

END {
        for (i in result) {
                print result[i]
        }
}' "$FILE_A" "$FILE_B" "$FILE_C" | sort -k1 > "$OUTPUTFILE"
```

### Syntax

```bash
./04.log_merge.bash <Level 0 Parsing Log> <Convert Parsing Log> <Incre 1 Parsing Log> <Output File>
```

### Usage

```bash
./04.log_merge.bash /backup/level0_log_parsing.log /backup/rman_convert_log_parsing.log /backup/rman_inc1_log_parsing.log merged_log.log
```

### Description

각 파싱한 로그들을 합쳐 Incremental Level 1 백업에 대해 ASIS 파일, 파일번호 및 Convert 된 TOBE 파일명을 기록한다. 이런 스크립트를 사용하는 이유는 위에서 설명했듯, ASM 을 사용할 경우 기존 파일명이 바뀌게 되며, 또한 각 핸들별 포함된 데이터파일에 대해 TOBE 파일명을 확인하기 위함이다. 결과적으로 해당 스크립트는 “**핸들, 파일번호, TOBE 파일명**” 형식의 CSV 파일을 생성한다. 이제 이 병합된 로그를 통해 Incremental Level 1 백업을 적용하는 스크립트를 수행할 수 있다. 

만약 Incremental Level 1 백업을 여러차례 받을경우 세번째 인자만 그때그때 받은 Level 1 증분 백업의 로그를 사용하면 된다.

## (Linux) Convert Level 1 Backup Convert (`conv_l1.sh`)

**Script**

```bash
#!/bin/bash

if [ $# -ne 2 ]; then
	echo "Usage : $0 <Level 1 Backup Piece Location> <Script Output Location>"
	echo exit 1
fi

INPUT_LOC=$1
OUTPUT_FILE=$2

# Heading
{
	printf "SET SERVEROUTPUT ON\n"
	printf "DECLARE\n"
	printf " V_HANDLE      VARCHAR2(512);\n";
	printf " V_COMMENT     VARCHAR2(512);\n";
	printf " V_MEDIA       VARCHAR2(80);\n";
	printf " V_CONCUR      BOOLEAN;\n";
	printf " v_RECID       NUMBER;\n";
	printf " V_STAMP       NUMBER;\n";
	printf " V_PLATFRMTO   NUMBER := 6;\n";
	printf " V_SAME_ENDIAN NUMBER := 1;\n";
	printf " V_DEVTYPE      VARCHAR2(512);\n";
	printf "BEGIN\n";
} > "$OUTPUT_FILE"

ls -l $1 | egrep -v "drw|log|total" | awk -v LOC="$INPUT_LOC" '
{
	printf " BEGIN\n";
	printf "  V_DEVTYPE := SYS.DBMS_BACKUP_RESTORE.DEVICEALLOCATE;\n";
	printf "  SYS.DBMS_BACKUP_RESTORE.BACKUPBACKUPPIECE(\n";
	printf "   bpname => \047%s/%s\047,\n",LOC,$NF;
	printf "   fname  => \047%s/%s**_conv**\047,\n",LOC,$NF;
	printf "   handle => V_HANDLE, media => V_MEDIA, comment => V_COMMENT,\n";
	printf "   concur => V_CONCUR, recid => V_RECID, stamp => V_STAMP, check_logical => FALSE,\n";
	printf "   copyno => 1, deffmt => 0, copy_recid => 0, copy_stamp => 0, npieces => 1, \n";
	printf "   dest => 0, pltfrmfr => 6);\n";
	printf " EXCEPTION\n";
	printf "  WHEN OTHERS\n";
	printf "  THEN\n";
	printf "   DBMS_OUTPUT.PUT_LINE(\047ERROR IN CONVERSION\047||SQLERRM);\n";
	printf " END;\n";
	printf " SYS.DBMS_BACKUP_RESTORE.DEVICEALLOCATE;\n";
	printf " DBMS_OUTPUT.PUT_LINE(\047Handle : \047|| V_HANDLE);\n";
}

END {
  printf "END;\n";
  printf "/\n";
}' >> $OUTPUT_FILE"
```

### Syntax

```bash
./conv_l1.sh <Level 1 Backup Piece Location> <Script Output Location>
```

### Usage

```bash
./conv_l1.sh /ora_mig/inc1 lvl1_conv.sql
```

### Description

기존에 받은 Level 1 백업피스를 TOBE 용으로 Convert 를 수행하는 스크립트를 생성해 준다. 기존 Level 1 의 위치에 생성되며, 파일 이름 뒤에 _conv 가 추가되어 생성된다. 이 규칙은 다음 Level 1 복구 명령어 생성 스크립트에서 활용된다.

생성된 스크립트는 sysdba 권한으로 데이터베이스 접속 후 수행하면 된다.

## (Linux) Incre Level 1 Recover : `05.incre_apply_gen.bash`

### Script

```bash
#!/bin/bash

if [ $# -ne 2 ]; then
        echo "Usage : $0 <merged rman log> <output script>"
        exit 1
fi

INPUT_FILE=$1
OUTPUT_FILE=$2

# Heading
{
        printf "SET SERVEROUTPUT ON\n"
        printf "SET TRIMSPOOL ON\n"
        printf "SPOOL RECOV_1_SESSION.log\n"
        printf "DECLARE\n"
        printf " V_CHANNEL    VARCHAR2(512);\n"
        printf " V_OUTHANDLE  VARCHAR2(512);\n"
        printf " V_OUTTAG     VARCHAR2(30);\n"
        printf " V_DONE       BOOLEAN;\n"
        printf " V_FAILOVER   BOOLEAN;\n"
        printf "BEGIN\n"
} > "$OUTPUT_FILE"

awk -F ',' '
function print_footer(handle) {
        if (handle != "") {
                printf " SYS.DBMS_BACKUP_RESTORE.RESTORESETPIECE(\n";
                printf "   handle=>\047%s**_conv**\047,\n", handle;
                printf "   tag=>NULL,\n";
                printf "   fromdisk=>TRUE,\n";
                printf "   recid=>0,\n";
                printf "   stamp=>0);\n";
                printf " SYS.DBMS_BACKUP_RESTORE.RESTOREBACKUPPIECE(\n";
                printf "   done=>V_DONE,\n";
                printf "   params=>NULL,\n";
                printf "   outhandle=>V_OUTHANDLE,\n";
                printf "   outtag=>V_OUTTAG,\n";
                printf "   failover=>V_FAILOVER);\n";
                printf " SYS.DBMS_BACKUP_RESTORE.RESTORECANCEL(TRUE);\n";
                printf " SYS.DBMS_BACKUP_RESTORE.DEVICEDEALLOCATE;\n\n";
        }
}

{
        gsub(/^[ \t]+|[ \t]+$/, "", $1);
        gsub(/^[ \t]+|[ \t]+$/, "", $2);
        gsub(/^[ \t]+|[ \t]+$/, "", $3);

        # Handle Changed?
        if ($1 != current_handle) {
                # If Handle exists, print footer
                if (current_handle != "") {
                        print_footer(current_handle);
                }
                # New Handle header
                printf " V_CHANNEL := SYS.DBMS_BACKUP_RESTORE.DEVICEALLOCATE;\n";
                printf " SYS.DBMS_BACKUP_RESTORE.APPLYSETDATAFILE(\n";
                printf "   check_logical => FALSE, \n"
                printf "   cleanup => FALSE); \n";
                current_handle = $1;
        }

        # Apply Datafile loop
        printf " SYS.DBMS_BACKUP_RESTORE.APPLYDATAFILETO(\n";
        printf "   dfnumber=>%s,\n",$2;
        printf "   toname=>\047%s\047,\n",$3;
        printf "   fuzziness_hint=>0,\n";
        printf "   max_corrupt=>0,\n";
        printf "   islevel0=>0,\n";
        printf "   recid=>0,\n";
        printf "   stamp=>0 );\n";

}

END {
        print_footer(current_handle);
}' $INPUT_FILE >> "$OUTPUT_FILE"

{
printf "END;\n"
printf "/\n"
printf "exit/\n"
} >> "$OUTPUT_FILE"
```

### Syntax

```bash
./05.incre_apply_gen.bash <Merged Log> <Output Recover Script>
```

### Usage

```bash
./05.incre_apply_gen.bash merged_log.log incre_lvl1_recover.sql
```

### Description

Incre Level 1 백업을 적용하는 SQL 을 작성한다. 각 핸들별 적용하는 데이터파일들을 기록하고 반복적으로 복구수행을 하는 스크립트를 생성하는데, 아래 사항을 주의한다.

- 스크립트 중간 `printf "   handle=>\047%s**_conv**\047,\n", handle;` 부분을 상황에 맞게 수정한다.
    - `_conv` 부분은 기존의 Level 1 백업 핸들을 컨버트한 파일명이다.
    - 가급적 기존 Level 1 백업 핸들명이 `somehandlename` 이라면, Level 1 백업에 대하여 Linux 용으로 Convert 시 `somehandlename_conv` 으로 변환하였다.

이제 해당 스크립트를 수행한 결과 위 예시대로 `incre_lvl1_recover.sql` 파일이 생성된다. 해당 스크립트를 sysdba 권한으로 수행하면, Incre Level 1 시점으로 모든 데이터파일이 복구된다.

## (Linux) Parfile 생성 : `06.make_par.bash`

### Script

```bash
if [ $# -ne 2 ]; then
        echo "Usage : $0 <Level 0 Convert Log> <output parfile>"
        exit 1
fi

INPUT_FILE=$1
OUTPUT_FILE=$2

awk '
BEGIN {
print "dumpfile=XTTS_TBS_METADATA.dmp"
print "directory=XTTS_MIG_DIR"
print "logfile=XTTS_TBS_METADATA_IMP.log"
print "transport_datafile="
i=1
}

/converted datafile=/ {
match($0,/=[+AZaz/0-9.]+/)
filename=substr($0,RSTART+1)

if (i == 1) {
 print "\047",filename,"\047"
 i++
} else {
 print ",\047",filename,"\047"
}
}' $INPUT_FILE  > $OUTPUT_FILE
```

### Syntax

```bash
./06.make_par.bash <Level 0 Convert Log> <Output Log>
```

### Usage

```bash
./06.make_par.bash rman_convert_log xtts_import.par
```

### Description

최종적으로 TTS 메타를 Import 하기 위한 Datapump 파라메터 파일을 생성한다. 데이터파일의 개수가 적을 경우 직접 구성해도 문제가 없을 것이나, 상당히 많은 수의 데이터파일에 대하여 리스트업을 하려면 상당히 번거로운 작업이 될 수 있으므로, 기존 Level 0  Convert 를 수행한 RMAN 로그를 통해 간편하게 수행한다.

데이터펌프의 각 파라메터는 자신의 상황에 맞게 수정해 사용하면 된다.

<aside>
💡

만약 xTTS 도중 ASIS 에서 신규 데이터파일이 추가되었을 경우, 반드시 해당 데이터파일에 대한 Level 0 백업을 ToBE 에 별도로 구성하여야 한다.

</aside>
