---
title: "Migration Azure PostgreSQL from single to FLEX"
description: "Process of migration Azure POstgreSQL from Single to FLEX"
---
  IT has 3 phases

   Pre-migration 
    pre_migration.sh 
    
     Get Azure POstgreSQL single server 
       It will get  security DDL using pg_dumpall
       DB create scripts based on single server DB owner
       recordc , indexes , constraints count

       source ~/.bashrc
SCRIPTS=/u08/migration
strttm=$(psql -h $src_host -U $src_usr -d postgres -t -A -c " select cast(now() as timestamp ) ")
# get Roles and priviliges
pg_dumpall  -h $src_host -U $src_usr  --roles-only -f $SCRIPTS/${src_host}_privs.ddl

# generate create database scripts at target
psql -h $src_host -U $src_usr -d postgres -t -A -c " select ' create database '||rtrim(datname)||' with owner '||rtrim(pg_catalog.pg_get_userbyid(datdba))||';' from pg_database where datname not in ('template1','template0','azure_maintenance','azure_sys') "  -o $SCRIPTS/crt_db_at_target.sql

# get DB size at source

psql -h $src_host -U $src_usr -d postgres -c "\l+" > $SCRIPTS/${src_host}_dbsz.lst

for db in $(psql -h $src_host -U $src_usr  -t -A -c " select datname from pg_database  where datname not in ('template1','template0','azure_maintenance','azure_sys') " )
do

echo " database backup for database $db "

pg_dump -Fc -h $src_host  -U $src_usr  -d  $db  -f $SCRIPTS/pg_dump_${db}.dmp

echo " get roles and priviliges in all schemas in the database $db "

psql -h $src_host -U $src_usr -d  $db -t -A -c " select ' insert into src_roles_n_privs  values  ('||chr(39)||rtrim(pgr.rolname)||chr(39)||chr(44)||chr(39)||rtrim(pgc.relname)||chr(39)||chr(44)||chr(39)||rtrim(nsp.nspname)||chr(39)||chr(44)||chr(39)||rtrim(has_table_privilege(pgr.rolname, pgc.oid, 'SELECT')::text)||chr(39)||chr(44)||chr(39)||rtrim(has_table_privilege(pgr.rolname, pgc.oid, 'INSERT')::text)||chr(39)||chr(44)||chr(39)||rtrim(has_table_privilege(pgr.rolname, pgc.oid, 'UPDATE')::text)||chr(39)||chr(44)||chr(39)||rtrim(has_table_privilege(pgr.rolname, pgc.oid, 'DELETE')::text)||chr(39)||' );'  FROM pg_roles pgr CROSS JOIN pg_class pgc JOIN pg_namespace nsp ON pgc.relnamespace = nsp.oid WHERE  nsp.nspname NOT IN ('pg_catalog', 'information_schema')     AND pgc.relkind = 'r' " -o $SCRIPTS/get_src_${db}_rolesNprivs.sql

echo " get record count  all schemas in the database $db "

psql -h $src_host -U $src_usr  -d $db   -t -A -c " select  ' select  '||chr(39)||'  insert into tbl_rec_src values ('||chr(39)||'||chr(39)||'||chr(39)||rtrim(schemaname)||chr(39)||'||chr(39)||chr(44)||chr(39)||'||chr(39)||ltrim(tablename)||chr(39)||'||chr(39)||chr(44)||rtrim(count(*)::text)||chr(41)||chr(59) from '||rtrim(schemaname)||'.'||ltrim(tablename)||'  ;' from pg_tables where schemaname not in ('pg_catalog','information_schema') "  -o  $SCRIPTS/mk_get_src_${db}_tbl_rc_cnt.sql
psql -h $src_host -U $src_usr -d $db   -t -A  -f   $SCRIPTS/mk_get_src_${db}_tbl_rc_cnt.sql  -o $SCRIPTS/get_src_${db}_tbl_rc_cnt.sql

echo " get  indexes detail  for all schemas in the database $db "

psql -h $src_host -U $src_usr  -d $db   -t -A -c "select ' insert into get_src_idx_dtl values ('||chr(39)||rtrim(schemaname)||chr(39)||chr(44)||chr(39)||rtrim(tablename)||chr(39)||chr(44)||rtrim(count(indexname)::text)||');' from pg_indexes  group by schemaname,tablename ;"  -o $SCRIPTS/get_src_${db}_idx_dtl.sql

echo " get constraints details for all schemas in the database $db "

psql -h $src_host -U $src_usr  -d $db  -t -A -c "select ' insert into get_src_const_dtl values ('||chr(39)||rtrim( connamespace::regnamespace::text)||chr(39)||chr(44)||chr(39)||rtrim(conrelid::regclass::text)||chr(39)||chr(44)||rtrim(count(conname)::text)||');' from  pg_constraint  where connamespace::regnamespace::text  not in ('pg_catalog','information_schema') group by connamespace::regnamespace::text,conrelid::regclass::text ;" -o $SCRIPTS/get_src_${db}_const_dtl.sql

echo "             "
echo " ****************  end of pre migration activity for database $db  *************************** "
done

endtm=$(psql -h $src_host -U $src_usr -d postgres -t -A -c " select cast(now() as timestamp )")

jbtm=$(psql -h $src_host -U $src_usr -d postgres -t -A -c " select cast('$endtm' as timestamp ) - cast('$strttm' as timestamp )")
echo "***************************************************************************************"
echo " Pre Migration activity started  at                 : $strttm "
echo " Pre Migration activity ended    at                 : $endtm  "
echo "---------------------------------------------------------------------------------------"
echo " Duration of Pre Migration Activity is              : $jbtm "
echo "****************************************************************************************"



 migration.sh
  will create  run create user and roles 
  create DB
  restore database from single server pg_dump

  source ~/.bashrc
SCRIPTS=$(pwd)
strttm=$(psql -h $trg_host -U $trg_usr -d postgres -t -A -c " select cast(now() as timestamp ) ")

echo "  Create Roles and grant  priviliges at $trg_host  cluster "

psql -h $trg_host -U $trg_usr  -d postgres -f $SCRIPTS/${src_host}_privs.ddl -o $SCRIPTS/${src_host}_privs.log

echo " Drop database , if created "
psql -h $trg_host -U $trg_usr -d postgres -t -A -c " select ' drop database '||rtrim(datname)||';' from pg_database where datname not in ('template1','template0','azure_maintenance','azure_sys','postgres') "   -o $SCRIPTS/drp_db.sql

psql -h $trg_host -U $trg_usr -d postgres -f $SCRIPTS/drp_db.sql  -o $SCRIPTS/drp_db.log

echo " Re-created database  using scripts extracted in pre migration activity "

psql -h $trg_host -U $trg_usr -d postgres -f $SCRIPTS/crt_db_at_target.sql -o $SCRIPTS/crt_db_at_target.log


for db in $(psql -h $trg_host -U $trg_usr  -t -A -c " select datname from pg_database  where datname not in ('template1','template0','azure_maintenance','azure_sys') " )
do

echo " Restore database $db  from  backup took from source cluster $src_host  "

pg_restore -v -h $trg_host  -U $trg_usr   --no-owner --role=$trg_usr -d  $db  pg_dump_${db}.dmp

echo " create validation tables at target $trg_host "
psql -h $trg_host -U $trg_usr -d $db -f $SCRIPTS/crt_validate_tbl.ddl -o $SCRIPTS/crt_validate_tbl.log

psql -h $trg_host -U $trg_usr  -d $db -f $SCRIPTS/${src_host}_privs.ddl -o $SCRIPTS/${src_host}_privs.log

echo " load  data from source into target for validation "
 psql -h $trg_host -U $trg_usr -d $db  -f  $SCRIPTS/get_src_${db}_rolesNprivs.sql -o $SCRIPTS/get_src_${db}_rolesNprivs.log
 psql -h $trg_host -U $trg_usr -d $db  -f  $SCRIPTS/get_src_${db}_tbl_rc_cnt.sql  -o $SCRIPTS/get_src_${db}_tbl_rc_cnt.log
 psql -h $trg_host -U $trg_usr -d $db  -f  $SCRIPTS/get_src_${db}_idx_dtl.sql     -o $SCRIPTS/get_src_${db}_idx_dtl.log
 psql -h $trg_host -U $trg_usr -d $db  -f  $SCRIPTS/get_src_${db}_const_dtl.sql   -o $SCRIPTS/get_src_${db}_const_dtl.log
 psql -h $trg_host -U $trg_usr -d $db  -f $SCRIPTS/src_${db}_tbl_own.sql          -o $SCRIPTS/src_${db}_tbl_own.log

echo " get roles and priviliges in all schemas in the database $db "

psql -h $trg_host -U $trg_usr -d  $db -t -A -c " select ' insert into trg_roles_n_privs  values  ('||chr(39)||rtrim(pgr.rolname)||chr(39)||chr(44)||chr(39)||rtrim(pgc.relname)||chr(39)||chr(44)||chr(39)||rtrim(nsp.nspname)||chr(39)||chr(44)||chr(39)||rtrim(has_table_privilege(pgr.rolname, pgc.oid, 'SELECT')::text)||chr(39)||chr(44)||chr(39)||rtrim(has_table_privilege(pgr.rolname, pgc.oid, 'INSERT')::text)||chr(39)||chr(44)||chr(39)||rtrim(has_table_privilege(pgr.rolname, pgc.oid, 'UPDATE')::text)||chr(39)||chr(44)||chr(39)||rtrim(has_table_privilege(pgr.rolname, pgc.oid, 'DELETE')::text)||chr(39)||' );'  FROM pg_roles pgr CROSS JOIN pg_class pgc JOIN pg_namespace nsp ON pgc.relnamespace = nsp.oid WHERE  nsp.nspname NOT IN ('pg_catalog', 'information_schema')     AND pgc.relkind = 'r' " -o $SCRIPTS/get_trg_${db}_rolesNprivs.sql

psql -h $trg_host -U $trg_usr -d  $db -f  $SCRIPTS/get_trg_${db}_rolesNprivs.sql -o $SCRIPTS/get_trg_${db}_rolesNprivs.log

echo " get record count  all schemas in the database $db "

psql -h $trg_host -U $trg_usr  -d $db   -t -A -c " select  ' select  '||chr(39)||'  insert into tbl_rec_src values ('||chr(39)||'||chr(39)||'||chr(39)||rtrim(schemaname)||chr(39)||'||chr(39)||chr(44)||chr(39)||'||chr(39)||ltrim(tablename)||chr(39)||'||chr(39)||chr(44)||rtrim(count(*)::text)||chr(41)||chr(59) from '||rtrim(schemaname)||'.'||ltrim(tablename)||'  ;' from pg_tables where schemaname not in ('pg_catalog','information_schema') "  -o  $SCRIPTS/mk_get_trg_${db}_tbl_rc_cnt.sql
psql -h $trg_host -U $trg_usr -d $db   -t -A  -f   $SCRIPTS/mk_get_trg_${db}_tbl_rc_cnt.sql  -o $SCRIPTS/get_trg_${db}_tbl_rc_cnt.sql

psql -h $trg_host -U $trg_usr  -d $db -f  $SCRIPTS/get_trg_${db}_tbl_rc_cnt.sql -o $SCRIPTS/get_trg_${db}_tbl_rc_cnt.log

echo " get  indexes detail  for all schemas in the database $db "

psql -h $trg_host -U $trg_usr  -d $db   -t -A -c "select ' insert into get_src_idx_dtl values ('||chr(39)||rtrim(schemaname)||chr(39)||chr(44)||chr(39)||rtrim(tablename)||chr(39)||chr(44)||rtrim(count(indexname)::text)||');' from pg_indexes  group by schemaname,tablename ;"  -o $SCRIPTS/get_trg_${db}_idx_dtl.sql

psql -h $trg_host -U $trg_usr  -d $db -f  $SCRIPTS/get_trg_${db}_idx_dtl.sql -o $SCRIPTS/get_trg_${db}_idx_dtl.log

echo " get constraints details for all schemas in the database $db "

psql -h $trg_host -U $trg_usr  -d $db  -t -A -c "select ' insert into get_src_const_dtl values ('||chr(39)||rtrim( connamespace::regnamespace::text)||chr(39)||chr(44)||chr(39)||rtrim(conrelid::regclass::text)||chr(39)||chr(44)||rtrim(count(conname)::text)||');' from  pg_constraint  where connamespace::regnamespace::text  not in ('pg_catalog','information_schema') group by connamespace::regnamespace::text,conrelid::regclass::text ;" -o $SCRIPTS/get_trg_${db}_const_dtl.sql

psql -h $trg_host -U $trg_usr  -d $db  -f $SCRIPTS/get_trg_${db}_const_dtl.sql -o $SCRIPTS/get_trg_${db}_const_dtl.log

echo "                                                                                               "
echo " ****************  end of migration activity for database $db  *************************** "
done

echo "                                                                                               "
echo " After restore database from source backup , get DB size "
psql -h $trg_host -U $trg_usr -d postgres -c " select datname , pg_size_pretty(pg_database_size(datname)) as db_size from pg_database" -o $SCRIPTS/get_aft_db_sz.lst

endtm=$(psql -h $trg_host -U $trg_usr -d postgres -t -A -c " select cast(now() as timestamp )")

jbtm=$(psql -h $trg_host -U $trg_usr -d postgres -t -A -c " select cast('$endtm' as timestamp ) - cast('$strttm' as timestamp )")
echo "***************************************************************************************"
echo " Migration activity started  at                 : $strttm "
echo " Migration activity ended    at                 : $endtm  "
echo "---------------------------------------------------------------------------------------"
echo " Duration of Migration Activity is              : $jbtm "
echo "****************************************************************************************"


Post migration
re-sert indentity column
validate objects , records , indexes and constraint , user and roles



source ~/.bashrc
SCRIPTS=/u08/migration
strttm=$(psql -h $trg_host -U $trg_usr -d postgres -t -A -c " select cast(now() as timestamp ) ")



for db in $(psql -h $trg_host -U $trg_usr  -t -A -c " select datname from pg_database  where datname not in ('template1','template0','azure_maintenance','azure_sys') " )
do

echo " reset sequences for all schemas in the database $db "

$SCRIPTS/reset_sequence.sh $trg_host $trg_usr $db


echo " run analyze  all tables in the database $db "
   psql -h $trg_host -U $trg_usr  -d $db  -t -A -c " select ' analyze '||rtrim(schemaname)||'.'||ltrim(tablename)||';' from pg_tables where schemaname not in ('pg_catalog','information_schema') " -o  $SCRIPTS/analyze-$db.sql
   psql -h  $trg_host  -U $trg_usr  -d $db  -f  $SCRIPTS/analyze-$db.sql  -o $SCRIPTS/analyze-$db.log

echo " default priviliges at source $src_host for database $db "

psql -h $src_host -U $scr_usr -d $db -E -c "\dpp"

echo " default priviliges at target $trg_host for database $db "

psql -h $trg_host -U $trg_usr -d $db -E -c "\dpp"


echo "                                                                                               "
echo " ****************  end of pre migration activity for database $db  *************************** "
done

echo "                                                                                               "
echo " After restore database from source backup , get DB size "
psql -h $trg_host -U $trg_usr -d postgres -c " select datname , pg_size_pretty(pg_database_size(datname)) as db_size from pg_database" -o $SCRIPTS/get_aft_db_sz.lst

echo " After migration all Databases sanitory check  details "
echo "----------------------------------------------------------------------------------------------------"
psql -h $trg_host -U $trg_usr -d postgres -f $SCRIPTS/get_db_healtch_chk.sql
echo "*********************** End of the DB sanitory check report ****************************************"


endtm=$(psql -h $trg_host -U $trg_usr -d postgres -t -A -c " select cast(now() as timestamp )")

jbtm=$(psql -h $trg_host -U $trg_usr -d postgres -t -A -c " select cast('$endtm' as timestamp ) - cast('$strttm' as timestamp )")
echo "***************************************************************************************"
echo " post migration activity   started  at                 : $strttm "
echo " post migration activity   ended    at                 : $endtm  "
echo "---------------------------------------------------------------------------------------"
echo " Duration of post Migration Activity              : $jbtm "
echo "****************************************************************************************"

