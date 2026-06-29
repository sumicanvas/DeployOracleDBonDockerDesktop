# Deploy OracleDB on Docker Desktop to test Korean Sample Data

# Oracle Docker 한글 샘플 데이터 적재 및 UTF-8 JSON Export 가이드

이 문서는 Docker Desktop에서 Oracle Free 데이터베이스를 실행하고, 한글 행정동 샘플 데이터를 CP949 파일로 변환한 뒤 SQL*Loader로 적재하고, 최종적으로 UTF-8 JSON 파일로 export하는 과정을 정리한 가이드입니다.

실제 비밀번호나 개인 계정 정보는 GitHub에 올리지 않도록 placeholder로 표기했습니다.

## 환경 정보

- OS: macOS Apple Silicon
- 컨테이너 런타임: Docker Desktop
- Oracle 이미지: `gvenzl/oracle-free:23-slim`
- Oracle DB 캐릭터셋: `AL32UTF8`
- 로딩 입력 파일 인코딩: CP949
- SQL*Loader 캐릭터셋 지정값: `KO16MSWIN949`
- JSON export 인코딩: UTF-8
- 샘플 스키마: `KOR_SAMPLE`
- 샘플 테이블: `KOR_SAMPLE.KR_ADMIN_AREA`

## 전체 흐름

1. Oracle Free Docker 이미지를 내려받습니다.
2. Oracle 컨테이너를 실행합니다.
3. Oracle DB 캐릭터셋을 확인합니다.
4. 한글 행정동 샘플 데이터를 다운로드합니다.
5. GeoJSON 데이터를 CSV로 변환합니다.
6. CSV 파일을 UTF-8에서 CP949로 변환합니다.
7. Oracle에 샘플 계정과 테이블을 생성합니다.
8. SQL*Loader로 CP949 파일을 로딩합니다.
9. 로딩된 한글 데이터를 검증합니다.
10. SQL*Plus로 UTF-8 JSON 파일을 export합니다.
11. streaming 방식으로도 JSON export를 수행합니다.

## 1. Oracle Free Docker 이미지 내려받기

```sh
docker pull gvenzl/oracle-free:23-slim
```

Docker CLI가 Docker Desktop이 아닌 다른 context를 바라보는 경우에는 context를 명시해서 실행합니다.

```sh
env -u DOCKER_HOST docker --context desktop-linux pull gvenzl/oracle-free:23-slim
```

## 2. Oracle 컨테이너 실행

```sh
docker run -d \
  --name oracle-free \
  -p 1521:1521 \
  -e ORACLE_PASSWORD='<ORACLE_ADMIN_PASSWORD>' \
  gvenzl/oracle-free:23-slim
```

접속 정보는 다음과 같습니다.

```text
Host: localhost
Port: 1521
Service name: FREEPDB1
Admin user: system
Admin password: <ORACLE_ADMIN_PASSWORD>
```

컨테이너 상태를 확인합니다.

```sh
docker ps --filter name=oracle-free
```

PDB 상태를 확인합니다.

```sh
docker exec oracle-free bash -lc 'printf "set heading off feedback off\nselect name, open_mode from v\$pdbs;\nexit\n" | sqlplus -s / as sysdba'
```

정상 상태라면 `FREEPDB1`이 `READ WRITE`로 표시됩니다.

```text
FREEPDB1  READ WRITE
```

## 3. Oracle DB 캐릭터셋 확인

현재 Oracle DB 자체의 캐릭터셋을 확인합니다.

```sh
docker exec oracle-free bash -lc 'printf "set heading off feedback off\nselect parameter, value from nls_database_parameters where parameter in ('"'"'NLS_CHARACTERSET'"'"','"'"'NLS_NCHAR_CHARACTERSET'"'"');\nexit\n" | sqlplus -s / as sysdba'
```

이번 환경에서는 다음과 같이 확인되었습니다.

```text
NLS_CHARACTERSET        AL32UTF8
NLS_NCHAR_CHARACTERSET  AL16UTF16
```

중요한 점은 DB 자체 캐릭터셋은 CP949가 아니라 `AL32UTF8`이라는 점입니다. 다만 SQL*Loader에서 입력 파일의 캐릭터셋을 `KO16MSWIN949`로 지정하면 CP949 파일을 올바르게 해석해서 UTF-8 DB에 적재할 수 있습니다.

## 4. 한글 샘플 데이터 다운로드

사용한 샘플 데이터는 `vuski/admdongkor` 저장소의 한글 행정동 데이터입니다.

```text
https://github.com/vuski/admdongkor
```

작업 디렉터리를 만들고 GeoJSON 파일을 다운로드합니다.

```sh
mkdir -p ./oracle-korean-sample

curl -L \
  "https://raw.githubusercontent.com/vuski/admdongkor/master/ver20260401/HangJeongDong_ver20260401.geojson" \
  -o ./oracle-korean-sample/HangJeongDong_ver20260401.geojson
```

이 데이터에는 시도명, 시군구명, 행정동명 등 한글 주소 정보가 포함되어 있습니다. 이번 적재에서는 총 `3558`건이 로딩되었습니다.

## 5. GeoJSON을 UTF-8 CSV로 변환

GeoJSON에서 필요한 속성만 추출해 CSV로 변환합니다.

```sh
node -e "const fs=require('fs'); const dir='./oracle-korean-sample'; const src=dir+'/HangJeongDong_ver20260401.geojson'; const out=dir+'/kr_admin_area.utf8.csv'; const j=JSON.parse(fs.readFileSync(src,'utf8')); const cols=['adm_cd2','adm_cd','sido','sgg','sidonm','sggnm','adm_nm']; const q=v=>'\"'+String(v??'').replace(/\"/g,'\"\"')+'\"'; const rows=j.features.map(f=>cols.map(c=>q(f.properties[c])).join(',')); fs.writeFileSync(out, rows.join('\n')+'\n', 'utf8'); console.log(rows.length);"
```

생성되는 파일은 UTF-8 CSV입니다.

```text
./oracle-korean-sample/kr_admin_area.utf8.csv
```

## 6. UTF-8 CSV를 CP949로 변환

SQL*Loader에서 CP949 입력 파일을 테스트하기 위해 CSV 파일을 CP949로 변환합니다.

```sh
iconv -f UTF-8 -t CP949 \
  ./oracle-korean-sample/kr_admin_area.utf8.csv \
  > ./oracle-korean-sample/kr_admin_area.cp949.csv
```

CP949 파일이 정상적으로 다시 UTF-8로 디코딩되는지도 확인합니다.

```sh
iconv -f CP949 -t UTF-8 ./oracle-korean-sample/kr_admin_area.cp949.csv > /dev/null
```

오류가 없다면 CP949 변환이 정상적으로 된 것입니다.

## 7. Oracle 샘플 계정 및 테이블 생성

샘플 데이터는 `SYSTEM` 계정에 직접 넣지 않고 별도 계정인 `KOR_SAMPLE`에 넣었습니다.

`create_schema.sql` 파일을 생성합니다.

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

컨테이너 안에 작업 디렉터리를 만들고 SQL 파일을 복사합니다.

```sh
docker exec oracle-free mkdir -p /tmp/oracle-korean-sample

docker cp ./oracle-korean-sample/create_schema.sql \
  oracle-free:/tmp/oracle-korean-sample/create_schema.sql
```

SQL 파일을 실행합니다.

```sh
docker exec oracle-free bash -lc \
  'sqlplus -s / as sysdba @/tmp/oracle-korean-sample/create_schema.sql'
```

## 8. SQL*Loader control file 생성

`load_admin_area.ctl` 파일을 생성합니다.

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

가장 중요한 설정은 아래 줄입니다.

```text
characterset KO16MSWIN949
```

이 설정은 SQL*Loader에게 입력 파일이 CP949 계열 한글 인코딩임을 알려줍니다. Oracle은 이 입력값을 DB 캐릭터셋인 `AL32UTF8`에 맞게 변환해 저장합니다.

control file 설정만 확인하려면 다음 명령을 사용할 수 있습니다.

```sh
docker exec oracle-free cat /tmp/oracle-korean-sample/load_admin_area.ctl
```

캐릭터셋 줄만 확인하려면 다음 명령을 사용합니다.

```sh
docker exec oracle-free grep -i characterset /tmp/oracle-korean-sample/load_admin_area.ctl
```

## 9. SQL*Loader로 CP949 데이터 로딩

control file과 CP949 CSV 파일을 컨테이너에 복사합니다.

```sh
docker cp ./oracle-korean-sample/load_admin_area.ctl \
  oracle-free:/tmp/oracle-korean-sample/load_admin_area.ctl

docker cp ./oracle-korean-sample/kr_admin_area.cp949.csv \
  oracle-free:/tmp/oracle-korean-sample/kr_admin_area.cp949.csv
```

SQL*Loader를 실행합니다.

```sh
docker exec oracle-free bash -lc \
  'sqlldr userid=system/<ORACLE_ADMIN_PASSWORD>@//127.0.0.1:1521/FREEPDB1 control=/tmp/oracle-korean-sample/load_admin_area.ctl log=/tmp/oracle-korean-sample/load_admin_area.log bad=/tmp/oracle-korean-sample/load_admin_area.bad'
```

성공하면 다음과 유사한 메시지가 출력됩니다.

```text
Table KOR_SAMPLE.KR_ADMIN_AREA:
  3558 Rows successfully loaded.
```

## 10. 적재 데이터 검증

로딩된 행 수와 한글 컬럼을 확인합니다.

```sh
docker exec oracle-free bash -lc 'printf "set pages 100 lines 220 feedback off\ncolumn adm_cd2 format a12\ncolumn sidonm format a20\ncolumn sggnm format a24\ncolumn adm_nm format a50\nselect count(*) as row_count from KOR_SAMPLE.KR_ADMIN_AREA;\nselect adm_cd2, sidonm, sggnm, adm_nm from KOR_SAMPLE.KR_ADMIN_AREA where rownum <= 5;\nexit\n" | sqlplus -s system/<ORACLE_ADMIN_PASSWORD>@//127.0.0.1:1521/FREEPDB1'
```

예상 결과는 다음과 같습니다.

```text
ROW_COUNT
---------
3558

ADM_NM
--------------------------------------------------
서울특별시 광진구 광장동
서울특별시 광진구 자양1동
서울특별시 광진구 자양2동
```

## 11. SQL*Plus 접속 방법

컨테이너 안의 SQL*Plus를 사용하는 방법입니다.

```sh
docker exec -it oracle-free sqlplus KOR_SAMPLE/<KOR_SAMPLE_PASSWORD>@//127.0.0.1:1521/FREEPDB1
```

로컬에 SQL*Plus를 설치했다면 다음처럼 접속할 수 있습니다.

```sh
sqlplus KOR_SAMPLE/<KOR_SAMPLE_PASSWORD>@//localhost:1521/FREEPDB1
```

접속 후 테스트 쿼리입니다.

```sql
select count(*) from kr_admin_area;

select adm_nm
from kr_admin_area
where rownum <= 5;
```

## 12. macOS Apple Silicon에 SQL*Plus 설치

컨테이너 안에는 SQL*Plus가 이미 포함되어 있습니다. 로컬 macOS에서도 직접 접속하고 싶다면 Oracle Instant Client ARM64 패키지를 설치합니다.

공식 패키지를 다운로드합니다.

```sh
curl -L \
  "https://download.oracle.com/otn_software/mac/instantclient/2326100/instantclient-basic-macos.arm64-23.26.1.0.0.dmg" \
  -o ./instantclient-basic-macos.arm64-23.26.1.0.0.dmg

curl -L \
  "https://download.oracle.com/otn_software/mac/instantclient/2326100/instantclient-sqlplus-macos.arm64-23.26.1.0.0.dmg" \
  -o ./instantclient-sqlplus-macos.arm64-23.26.1.0.0.dmg
```

SHA256 체크섬을 확인합니다.

```sh
shasum -a 256 ./instantclient-basic-macos.arm64-23.26.1.0.0.dmg
shasum -a 256 ./instantclient-sqlplus-macos.arm64-23.26.1.0.0.dmg
```

공식 페이지 기준 체크섬은 다음과 같습니다.

```text
5dc67a7e1cccd0a01d5bf53d7cf13b56f00999e3c2c1a309d8600cd766d80b41  instantclient-basic-macos.arm64-23.26.1.0.0.dmg
db973a9d2a672b462333feda091c8b2e2defe7aa2a2f4b266a8f36ee356d979e  instantclient-sqlplus-macos.arm64-23.26.1.0.0.dmg
```

DMG를 마운트하고 설치 스크립트를 실행합니다.

```sh
hdiutil attach ./instantclient-basic-macos.arm64-23.26.1.0.0.dmg
hdiutil attach ./instantclient-sqlplus-macos.arm64-23.26.1.0.0.dmg

sh /Volumes/instantclient-basic-macos.arm64-23.26.1.0.0/install_ic.sh
```

`sqlplus`를 PATH에 추가합니다.

```sh
ln -sf "$HOME/Downloads/instantclient_23_26/sqlplus" /opt/homebrew/bin/sqlplus
```

설치 버전을 확인합니다.

```sh
sqlplus -V
```

예상 출력입니다.

```text
SQL*Plus: Release 23.26.1.0.0 - Production
```

로컬 SQL*Plus 접속 시 `ORA-24454: client host name is not set` 오류가 발생하면 `/etc/hosts`에 현재 hostname을 추가합니다.

```sh
sudo sh -c 'printf "\n127.0.0.1 $(hostname)\n" >> /etc/hosts'
```

## 13. UTF-8 JSON 파일로 Export

Oracle에 저장된 데이터를 JSON 배열 파일로 export합니다. 이때 SQL*Plus 클라이언트 출력 인코딩을 UTF-8로 지정하기 위해 `NLS_LANG=KOREAN_KOREA.AL32UTF8`을 사용합니다.

`export_kr_admin_area_json_utf8.sql` 파일을 생성합니다.

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

SQL 파일을 컨테이너에 복사합니다.

```sh
docker cp ./oracle-korean-sample/export_kr_admin_area_json_utf8.sql \
  oracle-free:/tmp/oracle-korean-sample/export_kr_admin_area_json_utf8.sql
```

UTF-8 출력 설정을 적용해서 export를 실행합니다.

```sh
docker exec oracle-free bash -lc \
  'NLS_LANG=KOREAN_KOREA.AL32UTF8 sqlplus -L -s KOR_SAMPLE/<KOR_SAMPLE_PASSWORD>@//127.0.0.1:1521/FREEPDB1 @/tmp/oracle-korean-sample/export_kr_admin_area_json_utf8.sql'
```

생성된 JSON 파일을 호스트로 복사합니다.

```sh
docker cp oracle-free:/tmp/oracle-korean-sample/kr_admin_area.utf8.json \
  ./kr_admin_area.utf8.json
```

핵심 설정은 다음입니다.

```sh
NLS_LANG=KOREAN_KOREA.AL32UTF8
```

이 설정으로 SQL*Plus 출력이 UTF-8로 생성됩니다.

## 14. Streaming 방식으로 UTF-8 JSON Export

컨테이너 내부에 JSON 파일을 만들지 않고, SQL*Plus 출력(stdout)을 바로 로컬 파일로 저장할 수도 있습니다.

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

이 방식의 장점은 중간 JSON 파일을 컨테이너 안에 만들 필요가 없다는 점입니다.

## 15. JSON 파일 검증

파일 인코딩을 확인합니다.

```sh
file -I kr_admin_area.stream.utf8.json
```

예상 출력입니다.

```text
application/json; charset=utf-8
```

UTF-8 디코딩과 JSON 파싱이 정상인지 확인합니다.

```sh
node -e "const fs=require('fs'); const buf=fs.readFileSync('kr_admin_area.stream.utf8.json'); const txt=new TextDecoder('utf-8',{fatal:true}).decode(buf); const data=JSON.parse(txt); console.log(JSON.stringify({rows:data.length, first:data[0], last:data[data.length-1]}, null, 2));"
```

예상 결과는 다음과 같습니다.

```text
rows: 3558
```

## 최종 결과

생성된 주요 결과 파일은 다음과 같습니다.

```text
kr_admin_area.utf8.json
kr_admin_area.stream.utf8.json
```

이번 작업에서 확인한 핵심 포인트는 다음과 같습니다.

- Oracle DB 자체 캐릭터셋은 `AL32UTF8`입니다.
- 원본 데이터는 UTF-8 GeoJSON입니다.
- 로딩 테스트를 위해 CSV를 CP949로 변환했습니다.
- SQL*Loader에서 `characterset KO16MSWIN949`를 지정해 CP949 파일을 정상 로딩했습니다.
- JSON export 시 `NLS_LANG=KOREAN_KOREA.AL32UTF8`을 지정해 UTF-8 JSON을 생성했습니다.
- streaming export는 `docker exec -i ... > output.json` 방식으로 수행할 수 있습니다.

## 보안 주의사항

- 실제 Oracle 계정 비밀번호를 GitHub에 커밋하지 마세요.
- `<ORACLE_ADMIN_PASSWORD>`와 `<KOR_SAMPLE_PASSWORD>`는 예시용 placeholder입니다.
- 운영 환경에서는 환경변수, secret manager, vault 등을 사용하세요.
- 샘플 계정 `KOR_SAMPLE`에는 `create session`, `create table` 권한만 부여했습니다.
