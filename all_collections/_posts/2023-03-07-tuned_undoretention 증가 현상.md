---
layout: post
title: tuned_undoretention 증가 현상
date: 2023-03-07
categories: ["Oracle"]
---

UNDO TABLESPACE가 AUTO MANAGEMENT로 유지되는데 성능 테스트로 인해 Heavy Transaction 발생 시 tuned_undoretention이 지속적으로 증가하는 현상

### SYMPTOMS

> - UNDO Tablespace를 AUTOEXTENSIBLE = NO 상태로 생성하고 MAXBYTES 값을 설정하지 않음
> - UNDO가 AUTOEXTENSIBLE = NO 인 경우 UNDO MANAGEMENT를 AUTO로 설정한 상태에서 과도한 트랜잭션 발생시 TUNED_UNDORETENTION이 지속적으로 증가하는 이슈가 있음
> - Doc 1112431.1 - Undo Remains Unexpired When Using Non-autoextensible Datafiles For The Undo Tablespace
> - Bug 9681444 : TUNED_UNDORETENTION CAN BE TOO HIGH AFTER DB BOUNCE IF HIGH WORKLOAD BEFORE



확인쿼리

```sql

set linesize 200
set pagesize 200
set trimspool on
column totalsize heading "TOTAL_SIZE(M)"
column usedsize heading "USED_SIZE(M)"
column freesize heading "FREE_SIZE(M)"
column tablespace_name format a20
column totalsize format 999,999.9
column usedsize format 999,999.9
column freesize format 999,999.9
column used_percent format 999.9
column status format a12
column logging format a12
column extent_management format a12
column segment_space_management format a12
column retention format a12
select a.tablespace_name, a.totalsize,
       nvl(b.usedsize,0) usedsize,
       nvl(round(((b.usedsize/a.totalsize)*100),1),0) Used_Percent,
       c.freesize,
       d.status,
       d.logging,
       d.extent_management,
       d.segment_space_management,
       d.retention
  from (select tablespace_name, sum(bytes)/1024/1024 totalsize
          from dba_data_files
         group by tablespace_name
        union all
        select tablespace_name, sum(bytes)/1024/1024 totalsize
          from dba_temp_files
         group by tablespace_name) a,
       (select tablespace_name, sum(bytes)/1024/1024 usedsize
          from dba_segments
         group by tablespace_name) b,
       (select tablespace_name, sum(bytes)/1024/1024 freesize
          from dba_free_space
         group by tablespace_name) c,
       (select tablespace_name, status, logging,
               extent_management, segment_space_management, retention
          from dba_tablespaces) d
where d.tablespace_name = a.tablespace_name(+)
   and d.tablespace_name = b.tablespace_name(+)
   and d.tablespace_name = c.tablespace_name(+)
   and d.tablespace_name like '%UNDO%'
order by 1;

TABLESPACE_NAME      TOTAL_SIZE(M) USED_SIZE(M) USED_PERCENT FREE_SIZE(M) STATUS       LOGGING      EXTENT_MANAG SEGMENT_SPAC RETENTION
-------------------- ------------- ------------ ------------ ------------ ------------ ------------ ------------ ------------ ------------
UNDOTBS1                     200.0         83.4         41.7        115.6 ONLINE       LOGGING      LOCAL        MANUAL       NOGUARANTEE
UNDOTBS2                     200.0          1.3           .6        197.8 ONLINE       LOGGING      LOCAL        MANUAL       NOGUARANTEE
UNDOTBS3                     200.0          1.3           .6        197.8 ONLINE       LOGGING      LOCAL        MANUAL       NOGUARANTEE
```

```sql

col parameter for a50
col value for a10
col description for a50
set linesize 200
SELECT a.ksppinm "Parameter",
       b.ksppstvl "Value",
       a.ksppdesc "Description"
FROM x$ksppi a, x$ksppsv b
WHERE a.indx=b.indx and a.ksppinm='_undo_autotune'
ORDER BY a.ksppinm;
 
Parameter                                          Value      Description
-------------------------------------------------- ---------- --------------------------------------------------
_undo_autotune                                     TRUE       enable auto tuning of undo_retention
```

##### UNDO 영역 유지에 관련된 것

- NOGUARANTEE : 언두 유지 기능이 언두 테이블스페잉스 크기에 종송된다. 공간이 부족할 경우 언두 익스텐트 훔치기가 발생한다. 기본값이다.
- GUARANTEE   : 언두 유지 기능을 엄격히 적용하는 옵션

### SOLUTION

> 방안1) undo_autotune=false 설정 후 UNDO를 Manual 관리
>
> ex) alter system set "_undo_autotune"=false scope=both sid='*';
>
> 방안2) UNDO tablespace의 MAXSIZE를 actual size로 설정하고 autoextensible로 생성 혹은 _smu_debug_mode=33554432로 설정
>
> ex) ALTER DATABASE DATAFILE '+DATA/rac/datafile/undotbs1.256.861848649' AUTOEXTEND ON MAXSIZE 10G;
