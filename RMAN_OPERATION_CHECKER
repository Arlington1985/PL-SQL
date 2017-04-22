SET PAGESIZE 50000
SET LINESIZE 32000
set echo off
set serveroutput on
set trimout on
set tab off
ALTER session SET nls_date_format='dd.mm.yyyy hh24:mi:ss';

DECLARE
  l_incident_ticket_time varchar2(100);
  l_incident_delay_interval number;
  l_error_list varchar2(10000);
  l_error_behv varchar2(10000);
  l_operation_exp_period number;
  l_warning_as_error boolean :=FALSE;
  --l_oracle_version varchar2(1000);
  l_rc_version varchar2(1000);
  l_rc_owner varchar2(1000);
  l_rc_database varchar2(1000);
  l_count number;
   
  l_incident_start_time date;
 
  l_find_before number :=0;
  l_find_before_failed number :=0;
  l_find_before_succeed number :=0;
  l_find_before_running number :=0;
  l_find_after number :=0;
  l_find_after_failed number :=0;
  l_find_after_succeed number :=0;
  l_find_after_running number :=0;
  l_exit_status varchar2(1000) :='KO';
  l_exit_message varchar2(1000) :='Automata could not find any RMAN operation either before or after incident reported time. Ticket will be escalated';
  l_error_found_in_session number :=0;
  l_error_found_status varchar2(100);
BEGIN
  l_incident_ticket_time:='$incident_ticket_time';
  --l_incident_ticket_time:='06.03.2017 14:00:00';
  l_incident_delay_interval:=$incident_delay_interval;
  --l_incident_delay_interval:=10;
  l_error_list :='$error_list';
  --l_error_list :='RMAN-01009';
  l_error_behv :=$error_behv;  --CHECK|IGNORE
  --l_error_behv :='IGNORE';
  l_operation_exp_period :=$operation_exp_period;
  --l_operation_exp_period :=60;
  l_warning_as_error :=$warning_as_error;
  --l_warning_as_error :=FALSE;

  for c1 in (select banner, rownum rn from v$version)
  loop
    dbms_output.put_line(decode(c1.rn,1,'Oracle version: ','                ')||c1.banner);
  end loop;
  
  select count(0) into l_count from dba_objects where upper(object_name)='RCVER';
  if l_count>0 then
    select owner into l_rc_owner from dba_objects where upper(object_name)='RCVER' and rownum=1;

    execute immediate 'select version from '||l_rc_owner||'.rcver where rownum=1' into l_rc_version;
    dbms_output.put_line('Recovery Catalog version: '||l_oracle_version);
    
    execute immediate 'select database_name from '||l_rc_owner||'.rc_database where rownum=1' into l_rc_database;
    dbms_output.put_line('Recovery Catalog database name: '||l_rc_database);
    
  l_incident_start_time:=to_date(l_incident_ticket_time,'dd.mm.yyyy HH24:mi:ss')-1/24/60*l_incident_delay_interval;
 
  dbms_output.put_line(RPAD('=',100,'=')||'Input parameters'||RPAD('=',100,'='));   
  dbms_output.put_line('Incident ticket reported time: '||l_incident_ticket_time);
  dbms_output.put_line('Delay interval: '||l_incident_delay_interval);
  if l_error_behv is not null then
    dbms_output.put_line(l_error_behv||' this list: '||l_error_list);
  end if;
  dbms_output.put_line('Operation expired period: '||l_operation_exp_period);
  dbms_output.put_line('Define warning as error: '|| CASE l_warning_as_error WHEN TRUE then 'Yes' WHEN FALSE then 'No' else 'Not defined' END);
  dbms_output.put_line('Defined start time for the analysys: '||to_char(l_incident_start_time,'dd.mm.yyyy HH24:mi:ss'));   
  dbms_output.put_line(RPAD('=',100,'=')||'List of RMAN operations'||RPAD('=',100,'='));   
  dbms_output.put_line('>>>Start time for the analysys: '||to_char(l_incident_start_time,'dd.mm.yyyy HH24:mi:ss')||chr(10));
  dbms_output.put_line(
                         RPAD('No. ',4,' ')||
                         RPAD('Operation',25,' ')||
                         RPAD('Status',40,' ')||
                         RPAD('Start time',25,' ')||
                         RPAD('End time',25,' ')||
                         RPAD('Operation period',25,' ')||
                         RPAD('Input data size',20,' ')||
                         RPAD('Output data size',20,' ')||chr(10)
                      );
  for c1 in (
  SELECT
    sum_rn,
    sum_cnt,
    session_rn,
    session_cnt,
    recid,
    stamp,
    session_recid,
    start_time,
    end_time,
    row_type,
    operation||' '||NVL(object_type,'SESSION') operation,
    output_device_type,
    dbms_rcvman.Num2DisplaySize(input_bytes) input_bytes_display,
    dbms_rcvman.Num2DisplaySize(output_bytes) output_bytes_display,
    dbms_rcvman.Sec2DisplayTime(abs(end_time-start_time)*86400) time_taken_display,
    new_status status
    FROM
    (SELECT
        rs.*,op.session_recid ses_rec,
        (case
          when op.session_recid is null and rs.status in ('COMPLETED WITH ERRORS','COMPLETED WITH WARNINGS','FAILED') and l_error_behv='CHECK' then RPAD('COMPLETED(ERRORS ARE IGNORED)',40) --checklist
          when op.session_recid is null and rs.status in ('RUNNING WITH ERRORS') and l_error_behv='CHECK' then RPAD('RUNNING(ERRORS ARE IGNORED)',40) --checklist
          when op.session_recid is not null and rs.status in ('COMPLETED WITH ERRORS','COMPLETED WITH WARNINGS','FAILED') and l_error_behv='IGNORE' then RPAD('COMPLETED(ERRORS ARE IGNORED)',40) --ignorelist
          when op.session_recid is not null and rs.status in ('RUNNING WITH ERRORS') and l_error_behv='IGNORE' then RPAD('RUNNING(ERRORS ARE IGNORED)',40) --ignorelist
          else rs.status
        end) new_status,
        ROW_NUMBER() OVER (partition by rs.session_recid order by rs.start_time ASC) AS session_rn, 
        count(0) over (partition by rs.session_recid) as session_cnt,
        ROW_NUMBER() OVER (order by rs.start_time ASC) AS sum_rn, 
        count(0) over () as sum_cnt
      FROM
        V$RMAN_STATUS rs
        left join
        (with like_list as (SELECT LEVEL AS id, REGEXP_SUBSTR(l_error_list, '[^,]+', 1, LEVEL) AS data
   FROM dual
CONNECT BY REGEXP_SUBSTR(l_error_list, '[^,]+', 1, LEVEL) IS NOT NULL)
select distinct session_recid from v$rman_output o join
like_list l on (o.output like ('%ORA-'||l.data||'%') or o.output like ('%RMAN-'||l.data||'%'))) op
      on rs.session_recid=op.session_recid where rs.start_time>l_incident_start_time
      order by recid) order by sum_rn)
  loop
 
    if trim(c1.operation)!='RMAN' or (c1.session_cnt=1 and trim(c1.operation)='RMAN') then
      if c1.start_time<=to_date(l_incident_ticket_time,'dd.mm.yyyy HH24:mi:ss') then
        l_find_before:=l_find_before+1;
        if trim(c1.status) in ('FAILED','COMPLETED WITH ERRORS') or (l_warning_as_error=TRUE and c1.status in ('COMPLETED WITH WARNINGS')) then
          l_find_before_failed:= l_find_before_failed+1;
        elsif trim(c1.status) in ('COMPLETED','COMPLETED(ERRORS ARE IGNORED)') or (l_warning_as_error=FALSE and c1.status in ('COMPLETED WITH WARNINGS')) then
          l_find_before_succeed:= l_find_before_succeed+1;
        elsif trim(c1.status) in ('RUNNING WITH ERRORS','RUNNING WITH WARNINGS','RUNNING','RUNNING(ERRORS ARE IGNORED)') then
          l_find_before_running:= l_find_before_running+1;
        end if;
      else
        if l_find_after=0 then
          dbms_output.put_line('>>>Incident ticket reported time: '||l_incident_ticket_time||chr(10));
        end if;
        l_find_after:=l_find_after+1;
        if trim(c1.status) in ('FAILED','COMPLETED WITH ERRORS') or (l_warning_as_error=TRUE and c1.status in ('COMPLETED WITH WARNINGS')) then
          l_find_after_failed:= l_find_after_failed+1;
        elsif trim(c1.status) in ('COMPLETED','COMPLETED(ERRORS ARE IGNORED)') or (l_warning_as_error=FALSE and c1.status in ('COMPLETED WITH WARNINGS')) then
          l_find_after_succeed:= l_find_after_succeed+1;
        elsif trim(c1.status) in ('RUNNING WITH ERRORS','RUNNING WITH WARNINGS','RUNNING','RUNNING(ERRORS ARE IGNORED)') then
          l_find_after_running:= l_find_after_running+1;       
        end if;
      end if;
    end if;
   
    if c1.session_rn=1 then
      l_error_found_in_session :=0;
      l_error_found_status :=trim(c1.status);
    end if;
   
    if  not (trim(c1.status) in ('COMPLETED','RUNNING','COMPLETED(ERRORS ARE IGNORED)','RUNNING(ERRORS ARE IGNORED)') or (l_warning_as_error=FALSE and c1.status in ('COMPLETED WITH WARNINGS','RUNNING WITH WARNINGS')) ) then
        l_error_found_in_session :=1;
        l_error_found_status :=trim(c1.status);
    end if;
   
    dbms_output.put_line(
                         RPAD(c1.sum_rn,4,' ')||
                         RPAD(c1.operation,25,' ')||
                         RPAD(c1.status,40,' ')||
                         RPAD(to_char(c1.start_time,'dd.mm.yyyy HH24:mi:ss'),25,' ')||
                         RPAD(to_char(c1.end_time,'dd.mm.yyyy HH24:mi:ss'),25,' ')||
                         RPAD(c1.time_taken_display,25,' ')||
                         RPAD(c1.input_bytes_display,20,' ')||
                         RPAD(c1.output_bytes_display,20,' ')
                        );
   
    if c1.session_rn=c1.session_cnt then
      for c2 in (select output from V$RMAN_OUTPUT where session_recid=c1.session_recid and l_error_found_in_session=1 order by recid)            
      loop
        dbms_output.put_line('    '||c2.output);
      end loop;
      dbms_output.put_line(RPAD('-',200,'-'));
    end if;
   
    if c1.sum_rn=c1.sum_cnt then
      if l_error_found_in_session=0 then
        if (sysdate-c1.end_time)*24*60<l_operation_exp_period then
          l_exit_status:='OK';
          l_exit_message:='Automata was found that last RMAN session is in '||l_error_found_status||' status and not expired. Ticket will be resolved';
        else
          l_exit_status:='KO';
          l_exit_message:='Automata was found that last RMAN session is in '||l_error_found_status||' status, but was expired. Ticket will be escalated';
        end if;
      else
        l_exit_status:='KO';
        l_exit_message:='Automata was found that last RMAN session is in '||l_error_found_status||' status. Ticket will be escalated';
      end if;
    end if;
  end loop;
 
  --dbms_output.put_line('==================Analyzing database operation results=====================');   
  dbms_output.put_line(RPAD('=',100,'=')||'Analyzing database RMAN operations'||RPAD('=',100,'='));   

  if l_find_before=0 and l_find_after=0 then
    dbms_output.put_line('No any RMAN operation was found neither before nor after incident reported time');
  else
    dbms_output.put_line('Overall RMAN oerations was found: '||to_char(l_find_before+l_find_after)||chr(10));
    if l_find_before>0 then
      dbms_output.put_line(' RMAN operations was found before incident reported time: '||to_char(l_find_before));
      dbms_output.put_line('   - Failed operations: '||to_char(l_find_before_failed));
      dbms_output.put_line('   - Completed operations: '||to_char(l_find_before_succeed));
      dbms_output.put_line('   - Running operations: '||to_char(l_find_before_running));
    end if;
    if l_find_after>0 then
      dbms_output.put_line(' RMAN operations was found after incident reported time: '||to_char(l_find_after));
      dbms_output.put_line('   - Failed operations: '||to_char(l_find_after_failed));
      dbms_output.put_line('   - Completed operations: '||to_char(l_find_after_succeed));
      dbms_output.put_line('   - Running operations: '||to_char(l_find_after_running));
    end if;
  end if;
  else 
    l_exit_status:='KO';
    l_exit_message:='Automata was found that Recovery Catalog was not enabled. Ticket will be escalated.';
  end if;
--  dbms_output.put_line('=================================Summary==================================');   
  dbms_output.put_line(RPAD('=',100,'=')||'Summary'||RPAD('=',100,'='));   
  dbms_output.put_line('EXIT_STATUS:' ||l_exit_status);
  dbms_output.put_line('EXIT_MESSAGE:'||l_exit_message);
END;
/
