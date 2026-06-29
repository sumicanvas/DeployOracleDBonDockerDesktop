# Deploy OracleDB on Docker Desktop to test Korean Sample Data

Oracle Free 컨테이너를 Docker Desktop에서 실행하고, 한글 행정동 샘플 데이터를 CP949 파일로 변환해 SQL*Loader로 로딩한 뒤, UTF-8 JSON으로 export하는 절차를 정리한 문서입니다.

## Environment

- Host OS: macOS Apple Silicon
- Container runtime: Docker Desktop
- Oracle image: `gvenzl/oracle-free:23-slim`
- Oracle DB character set: `AL32UTF8`
- Input file encoding for load: CP949 / Oracle `KO16MSWIN949`
- Export encoding: UTF-8

## 1. Oracle Free Docker Image Pull

```sh
docker pull gvenzl/oracle-free:23-slim
```

If Docker CLI is pointing to a wrong context, explicitly use Docker Desktop context:

```sh
env -u DOCKER_HOST docker --context desktop-linux pull gvenzl/oracle-free:23-slim
```

## 2. Run Oracle Container

```sh
docker run -d \
  --name oracle-free \
  -p 1521:1521 \
  -e ORACLE_PASSWORD='<ORACLE_ADMIN_PASSWORD>' \
  gvenzl/oracle-free:23-slim
```

Connection info:

```text
Host: localhost
Port: 1521
Service name: FREEPDB1
Admin user: system
Admin password: <ORACLE_ADMIN_PASSWORD>
```

Check container status:

```sh
docker ps --filter name=oracle-free
```

Check Oracle PDB status:

```sh
docker exec oracle-free bash -lc 'printf "set heading off feedback off\nselect name, open_mode from v\$pdbs;\nexit\n" | sqlplus -s / as sysdba'
```

Expected status:

```text
FREEPDB1  READ WRITE
```

## 3. Check Database Character Set

```sh
docker exec oracle-free bash -lc 'printf "set heading off feedback off\nselect parameter, value from nls_database_parameters where parameter in ('"'"'NLS_CHARACTERSET'"'"','"'"'NLS_NCHAR_CHARACTERSET'"'"');\nexit\n" | sqlplus -s / as sysdba'
```

Expected result for this image:

```text
NLS_CHARACTERSET        AL32UTF8
NLS_NCHAR_CHARACTERSET  AL16UTF16
```

Note: The database itself is `AL32UTF8`. The input data is loaded from a CP949 file by telling SQL*Loader to interpret the input as `KO16MSWIN949`.

## 4. Download Korean Sample Data

Sample data source:

```text
https://github.com/vuski/admdongkor
```

Download GeoJSON:

```sh
mkdir -p ./oracle-korean-sample

curl -L \
  "https://raw.githubusercontent.com/vuski/admdongkor/master/ver20260401/HangJeongDong_ver20260401.geojson" \
  -o ./oracle-korean-sample/HangJeongDong_ver20260401.geojson
```

The source contains Korean administrative district names. In this run, `3558` rows were loaded.

## 5. Convert GeoJSON To CSV

Create a CSV file from selected GeoJSON properties:

```sh
node -e "const fs=require('fs'); const dir='./oracle-korean-sample'; const src=dir+'/HangJeongDong_ver20260401.geojson'; const out=dir+'/kr_admin_area.utf8.csv'; const j=JSON.parse(fs.readFileSync(src,'utf8')); const cols=['adm_cd2','adm_cd','sido','sgg','sidonm','sggnm','adm_nm']; const q=v=>'\"'+String(v??'').replace(/\"/g,'\"\"')+'\"'; const rows=j.features.map(f=>cols.map(c=>q(f.properties[c])).join(',')); fs.writeFileSync(out, rows.join('\n')+'\n', 'utf8'); console.log(rows.length);"
```

Convert UTF-8 CSV to CP949:

```sh
iconv -f UTF-8 -t CP949 \
  ./oracle-korean-sample/kr_admin_area.utf8.csv \
  > ./oracle-korean-sample/kr_admin_area.cp949.csv
```

Validate that the CP949 file can be decoded:

```sh
iconv -f CP949 -t UTF-8 ./oracle-korean-sample/kr_admin_area.cp949.csv > /dev/null
```

## 6. Create Oracle Schema And Table

Create `create_schema.sql`:

```sql
whenever sqlerror exit sql.sqlcode

alter session set container = FREEPDB1;

declare
  user_count number;
begin
  select count(*) into user_count from dba_users where username = 'KOR_SAMPLE';

  if user_count = 0 then
    execute immediate 'create user KOR_SAMPLE identified by "<KOR_SAMPLE_PASSWORD>" default tablespace USERS temporary tablespace TEMP quota unlimited on USERS';
  else
    execute immediate 'alter user KOR_SAMPLE identified by "<KOR_SAMPLE_PASSWORD>" account unlock';
    execute immediate 'alter user KOR_SAMPLE quota unlimited on USERS';
  end if;
end;
/

grant create session, create table to KOR_SAMPLE;

begin
  execute immediate 'drop table KOR_SAMPLE.KR_ADMIN_AREA purge';
exception
  when others then
    if sqlcode != -942 then
      raise;
    end if;
end;
/

create table KOR_SAMPLE.KR_ADMIN_AREA (
  adm_cd2 varchar2(10 char) primary key,
  adm_cd varchar2(8 char) not null,
  sido varchar2(2 char) not null,
  sgg varchar2(5 char) not null,
  sidonm varchar2(40 char) not null,
  sggnm varchar2(80 char) not null,
  adm_nm varchar2(160 char) not null
);

exit
```

Copy and execute it in the container:

```sh
docker exec oracle-free mkdir -p /tmp/oracle-korean-sample

docker cp ./oracle-korean-sample/create_schema.sql \
  oracle-free:/tmp/oracle-korean-sample/create_schema.sql

docker exec oracle-free bash -lc \
  'sqlplus -s / as sysdba @/tmp/oracle-korean-sample/create_schema.sql'
```

## 7. Load CP949 Data With SQL*Loader

Create `load_admin_area.ctl`:

```text
load data
characterset KO16MSWIN949
infile '/tmp/oracle-korean-sample/kr_admin_area.cp949.csv'
truncate
into table KOR_SAMPLE.KR_ADMIN_AREA
fields terminated by ',' optionally enclosed by '"'
trailing nullcols
(
  adm_cd2,
  adm_cd,
  sido,
  sgg,
  sidonm,
  sggnm,
  adm_nm
)
```

The important line is:

```text
characterset KO16MSWIN949
```

This tells SQL*Loader that the input file is CP949-compatible Korean text.

Copy load files into the container:

```sh
docker cp ./oracle-korean-sample/load_admin_area.ctl \
  oracle-free:/tmp/oracle-korean-sample/load_admin_area.ctl

docker cp ./oracle-korean-sample/kr_admin_area.cp949.csv \
  oracle-free:/tmp/oracle-korean-sample/kr_admin_area.cp949.csv
```

Run SQL*Loader:

```sh
docker exec oracle-free bash -lc \
  'sqlldr userid=system/<ORACLE_ADMIN_PASSWORD>@//127.0.0.1:1521/FREEPDB1 control=/tmp/oracle-korean-sample/load_admin_area.ctl log=/tmp/oracle-korean-sample/load_admin_area.log bad=/tmp/oracle-korean-sample/load_admin_area.bad'
```

Expected result:

```text
Table KOR_SAMPLE.KR_ADMIN_AREA:
  3558 Rows successfully loaded.
```

## 8. Verify Korean Data

```sh
docker exec oracle-free bash -lc 'printf "set pages 100 lines 220 feedback off\ncolumn adm_cd2 format a12\ncolumn sidonm format a20\ncolumn sggnm format a24\ncolumn adm_nm format a50\nselect count(*) as row_count from KOR_SAMPLE.KR_ADMIN_AREA;\nselect adm_cd2, sidonm, sggnm, adm_nm from KOR_SAMPLE.KR_ADMIN_AREA where rownum <= 5;\nexit\n" | sqlplus -s system/<ORACLE_ADMIN_PASSWORD>@//127.0.0.1:1521/FREEPDB1'
```

## 9. Install SQL*Plus On macOS ARM64

SQL*Plus is already available inside the Oracle container. If you also want local SQL*Plus on macOS Apple Silicon, install Oracle Instant Client ARM64 packages.

Download official packages:

```sh
curl -L \
  "https://download.oracle.com/otn_software/mac/instantclient/2326100/instantclient-basic-macos.arm64-23.26.1.0.0.dmg" \
  -o ./instantclient-basic-macos.arm64-23.26.1.0.0.dmg

curl -L \
  "https://download.oracle.com/otn_software/mac/instantclient/2326100/instantclient-sqlplus-macos.arm64-23.26.1.0.0.dmg" \
  -o ./instantclient-sqlplus-macos.arm64-23.26.1.0.0.dmg
```

Verify SHA256:

```sh
shasum -a 256 ./instantclient-basic-macos.arm64-23.26.1.0.0.dmg
shasum -a 256 ./instantclient-sqlplus-macos.arm64-23.26.1.0.0.dmg
```

Expected checksums:

```text
5dc67a7e1cccd0a01d5bf53d7cf13b56f00999e3c2c1a309d8600cd766d80b41  instantclient-basic-macos.arm64-23.26.1.0.0.dmg
db973a9d2a672b462333feda091c8b2e2defe7aa2a2f4b266a8f36ee356d979e  instantclient-sqlplus-macos.arm64-23.26.1.0.0.dmg
```

Mount and install:

```sh
hdiutil attach ./instantclient-basic-macos.arm64-23.26.1.0.0.dmg
hdiutil attach ./instantclient-sqlplus-macos.arm64-23.26.1.0.0.dmg

sh /Volumes/instantclient-basic-macos.arm64-23.26.1.0.0/install_ic.sh
```

Add SQL*Plus to PATH:

```sh
ln -sf "$HOME/Downloads/instantclient_23_26/sqlplus" /opt/homebrew/bin/sqlplus
```

Check version:

```sh
sqlplus -V
```

Expected version:

```text
SQL*Plus: Release 23.26.1.0.0 - Production
```

If local SQL*Plus fails with `ORA-24454: client host name is not set`, add your hostname to `/etc/hosts`:

```sh
sudo sh -c 'printf "\n127.0.0.1 $(hostname)\n" >> /etc/hosts'
```

## 10. Connect To Oracle

Using container SQL*Plus:

```sh
docker exec -it oracle-free sqlplus KOR_SAMPLE/<KOR_SAMPLE_PASSWORD>@//127.0.0.1:1521/FREEPDB1
```

Using local SQL*Plus:

```sh
sqlplus KOR_SAMPLE/<KOR_SAMPLE_PASSWORD>@//localhost:1521/FREEPDB1
```

Test query:

```sql
select count(*) from kr_admin_area;

select adm_nm
from kr_admin_area
where rownum <= 5;
```

## 11. Export JSON As UTF-8 File

Create `export_kr_admin_area_json_utf8.sql`:

```sql
set echo off
set feedback off
set heading off
set pagesize 0
set linesize 32767
set long 100000000
set longchunksize 32767
set trimspool on
set termout off
set verify off
set define off

spool /tmp/oracle-korean-sample/kr_admin_area.utf8.json
prompt [

select case when rn > 1 then ',' end || json_object(
         'adm_cd2' value adm_cd2,
         'adm_cd' value adm_cd,
         'sido' value sido,
         'sgg' value sgg,
         'sidonm' value sidonm,
         'sggnm' value sggnm,
         'adm_nm' value adm_nm
         returning varchar2(32767)
       )
from (
  select t.*, row_number() over (order by adm_cd2) rn
  from kor_sample.kr_admin_area t
)
order by adm_cd2;

prompt ]
spool off
exit
```

Run export with UTF-8 client encoding:

```sh
docker cp ./oracle-korean-sample/export_kr_admin_area_json_utf8.sql \
  oracle-free:/tmp/oracle-korean-sample/export_kr_admin_area_json_utf8.sql

docker exec oracle-free bash -lc \
  'NLS_LANG=KOREAN_KOREA.AL32UTF8 sqlplus -L -s KOR_SAMPLE/<KOR_SAMPLE_PASSWORD>@//127.0.0.1:1521/FREEPDB1 @/tmp/oracle-korean-sample/export_kr_admin_area_json_utf8.sql'

docker cp oracle-free:/tmp/oracle-korean-sample/kr_admin_area.utf8.json \
  ./kr_admin_area.utf8.json
```

The important setting is:

```sh
NLS_LANG=KOREAN_KOREA.AL32UTF8
```

This forces SQL*Plus output to UTF-8.

## 12. Export JSON With Streaming

This method does not create a JSON file inside the container. It streams SQL*Plus stdout directly to a local file.

```sh
docker exec -i oracle-free bash -lc 'NLS_LANG=KOREAN_KOREA.AL32UTF8 sqlplus -L -s KOR_SAMPLE/<KOR_SAMPLE_PASSWORD>@//127.0.0.1:1521/FREEPDB1' > kr_admin_area.stream.utf8.json <<'SQL'
set feedback off
set heading off
set pagesize 0
set linesize 32767
set long 100000000
set longchunksize 32767
set trimspool on
set verify off
set define off

prompt [

select case when rn > 1 then ',' end || json_object(
         'adm_cd2' value adm_cd2,
         'adm_cd' value adm_cd,
         'sido' value sido,
         'sgg' value sgg,
         'sidonm' value sidonm,
         'sggnm' value sggnm,
         'adm_nm' value adm_nm
         returning varchar2(32767)
       )
from (
  select t.*, row_number() over (order by adm_cd2) rn
  from kr_admin_area t
)
order by adm_cd2;

prompt ]
exit
SQL
```

## 13. Validate Exported JSON

Check file encoding:

```sh
file -I kr_admin_area.stream.utf8.json
```

Expected result:

```text
application/json; charset=utf-8
```

Validate UTF-8 decoding and JSON parsing:

```sh
node -e "const fs=require('fs'); const buf=fs.readFileSync('kr_admin_area.stream.utf8.json'); const txt=new TextDecoder('utf-8',{fatal:true}).decode(buf); const data=JSON.parse(txt); console.log(JSON.stringify({rows:data.length, first:data[0], last:data[data.length-1]}, null, 2));"
```

Expected row count:

```text
3558
```

## Summary

- Oracle DB character set is `AL32UTF8`.
- Source CSV was converted to CP949.
- SQL*Loader loaded CP949 correctly using `characterset KO16MSWIN949`.
- JSON export was generated as UTF-8 using `NLS_LANG=KOREAN_KOREA.AL32UTF8`.
- Streaming export is possible with `docker exec -i ... > output.json`.

## Security Notes

- Do not commit real Oracle account credentials or local passwords to GitHub.
- Replace `<ORACLE_ADMIN_PASSWORD>` and `<KOR_SAMPLE_PASSWORD>` with environment variables or secret manager values in production.
- The sample schema `KOR_SAMPLE` was granted only `create session` and `create table` privileges.

