# Oracle Migration Mysql
Migrate oracle row into MySQL insert statements as offline files


``` sql

-- The following script export some ORACLE tables into SQL insert statments 
-- It use MIGRATION_COLUMNS_SETTINGS, MIGRATION_TABELS_LIST table to change the columns names

BEGIN 
  DBMS_OUTPUT.PUT_LINE('STARTING');
END; 

/
-- Check if MIGRATION_COLUMNS_SETTINGS table exits or create it
DECLARE
nCount NUMBER;
v_sql LONG;

BEGIN
--DBMS_OUTPUT.PUT_LINE('Check MIGRATION_COLUMNS_SETTINGS table exists?');
SELECT count(*) into nCount FROM dba_tables where table_name = 'MIGRATION_COLUMNS_SETTINGS';
IF(nCount <= 0)
THEN
--  DBMS_OUTPUT.PUT_LINE('No, MIGRATION_COLUMNS_SETTINGS table not exists, Creating new one...');
  v_sql:='
  create table MIGRATION_COLUMNS_SETTINGS
  (
  ID NUMBER(3),
  TABLE_NAME VARCHAR2(30) NOT NULL,
  OLD_COLUMN_NAME VARCHAR2(30) NOT NULL,
  NEW_COLUMN_NAME VARCHAR2(30) NOT NULL
  )';
  execute immediate v_sql;
--ELSE
--DBMS_OUTPUT.PUT_LINE('Yes MIGRATION_COLUMNS_SETTINGS table  exists.');
END IF;

END;



--declare 
--  fHandle  UTL_FILE.FILE_TYPE;
--begin
--  fHandle := UTL_FILE.FOPEN('my_directory', 'test_file', 'w');
--
--  UTL_FILE.PUT(fHandle, 'This is the first line');
--  UTL_FILE.PUT(fHandle, 'This is the second line');
--  UTL_FILE.PUT_LINE(fHandle, 'This is the third line');
--
--  UTL_FILE.FCLOSE(fHandle);
--EXCEPTION
--  WHEN OTHERS THEN
--    DBMS_OUTPUT.PUT_LINE('Exception: SQLCODE=' || SQLCODE || '  SQLERRM=' || SQLERRM);
--    RAISE;
--end;


/
-- Check if MIGRATION_TABELS_LIST table exits or create it
DECLARE
nCount NUMBER;
v_sql LONG;

BEGIN
--DBMS_OUTPUT.PUT_LINE('Check MIGRATION_TABELS_LIST table exists?');
SELECT count(*) into nCount FROM dba_tables where table_name = 'MIGRATION_TABELS_LIST';
IF(nCount <= 0)
THEN
--  DBMS_OUTPUT.PUT_LINE('No, MIGRATION_TABELS_LIST table not exists, Creating new one...');
  v_sql:='
  create table MIGRATION_TABELS_LIST
  (
  ID NUMBER(3),
  TABLE_NAME VARCHAR2(30) NOT NULL
  )';
  execute immediate v_sql;
--ELSE
--DBMS_OUTPUT.PUT_LINE('Yes MIGRATION_TABELS_LIST table  exists.');
END IF;

END;


/


-- Create method to get new columns names
-- GET_COLUMN_NAME is function to get new column names from MIGRATION_COLUMNS_SETTINGS table
CREATE OR REPLACE FUNCTION  GET_COLUMN_NAME ( table_name  VARCHAR2, column_name  VARCHAR2   )
 RETURN READ_ONLY.MIGRATION_COLUMNS_SETTINGS.NEW_COLUMN_NAME%TYPE
  IS
    NEW_NAME READ_ONLY.MIGRATION_COLUMNS_SETTINGS.NEW_COLUMN_NAME%TYPE;
  BEGIN 
--   DBMS_OUTPUT.PUT_LINE('START SELECTING table_name:' || table_name || ' column_name:' || column_name );
    SELECT NEW_COLUMN_NAME
      INTO NEW_NAME
      FROM READ_ONLY.MIGRATION_COLUMNS_SETTINGS
      WHERE TABLE_NAME = table_name AND OLD_COLUMN_NAME = column_name;
      
      RETURN NEW_NAME;
      
      EXCEPTION
        WHEN OTHERS  THEN
          NEW_NAME := column_name;
--          DBMS_OUTPUT.PUT_LINE('Error SELECTING ' || 'An error was encountered - '||SQLCODE||' -ERROR- '||SQLERRM);
--          DBMS_OUTPUT.PUT_LINE('END SELECTING column_name:' ||  column_name || ' NEW_NAME:' || NEW_NAME );
          NEW_NAME := column_name;
          RETURN NEW_NAME;
   
END GET_COLUMN_NAME;
 
/


-- create path for migration

create or replace directory temp_dir as './your/path/here';
/
grant read, write on directory temp_dir to PUBLIC
/

-- Start migration
DECLARE
  cur SYS_REFCURSOR;
  curid NUMBER;
  desctab DBMS_SQL.desc_tab;
  colcnt NUMBER;
  namevar VARCHAR2(4000);
  numvar NUMBER;
  datevar DATE;
  total_row_fetched number:=0;
  file_name varchar(50):='';
  out_columns varchar2(10000);
  out_values varchar2(10000);
  fHandle  UTL_FILE.FILE_TYPE;
  batch_file_number NUMBER := 1;
  
BEGIN

  --  fHandle := UTL_FILE.FOPEN('temp_dir', 'out.sql', 'w');
  -- write to file with name 
  --  fHandle := UTL_FILE.FOPEN('temp_dir', 'out'|| TO_CHAR(sysdate, 'yyyy-mm-dd hh:mm' )||'-part-'|| batch_file_number ||'.sql', 'w');
  -- get list of tabels to log 
  FOR rec IN (SELECT TABLE_NAME 
                FROM MIGRATION_TABELS_LIST 
               ORDER BY TABLE_NAME)
  LOOP
     OPEN cur FOR 'SELECT * FROM '||rec.TABLE_NAME||' ORDER BY 1';
    DBMS_OUTPUT.put_line('Working on table:' ||rec.TABLE_NAME);
    
    
    
    
    
    
    curid := DBMS_SQL.to_cursor_number(cur);
    DBMS_SQL.describe_columns(curid, colcnt, desctab);

    out_columns := 'INSERT INTO '||rec.TABLE_NAME||'(';
    



    
    FOR indx IN 1 .. colcnt LOOP
    
      -- get table name and column name, check if they are exist in the MIGRATION_COLUMNS_SETTINGS
      -- if it exists get the new name, else use current name
      out_columns := out_columns||READ_ONLY.GET_COLUMN_NAME( rec.TABLE_NAME , desctab(indx).col_name )||',';
      IF desctab (indx).col_type = 2 
      THEN
        DBMS_SQL.define_column (curid, indx, numvar); 
      ELSIF desctab (indx).col_type = 12
      THEN
        DBMS_SQL.define_column (curid, indx, datevar); 
      ELSE
        DBMS_SQL.define_column (curid, indx, namevar, 4000); 
      END IF;
    END LOOP;

    out_columns := rtrim(out_columns,',')||') VALUES (';

    WHILE DBMS_SQL.fetch_rows (curid) > 0 
    LOOP
      out_values := '';
      FOR indx IN 1 .. colcnt 
      LOOP
        IF (desctab (indx).col_type = 1) 
        THEN
          DBMS_SQL.COLUMN_VALUE (curid, indx, namevar);
          out_values := out_values||''''||namevar||''',';
        ELSIF (desctab (indx).col_type = 2)
        THEN
          DBMS_SQL.COLUMN_VALUE (curid, indx, numvar);
          out_values := out_values||numvar||',';
        ELSIF (desctab (indx).col_type = 12)
        THEN
          DBMS_SQL.COLUMN_VALUE (curid, indx, datevar);
          out_values := out_values|| ''''||
            to_char(datevar,'YYYY-MM-DD HH24:MI:SS')||
             ''',';
        END IF;
      END LOOP; 
     DBMS_OUTPUT.put_line(out_columns||rtrim(out_values,',')||');');
     -- create new file for each 2M record
    -- write to file with name 
    if mod ( total_row_fetched, 2000 ) = 0 then
      /* close old file, open new */
      -- if file open close it
      if SYS.UTL_FILE.IS_OPEN(fHandle)then
       UTL_FILE.FCLOSE(fHandle);
      end if;
       
       
       file_name := 'out-'|| TO_CHAR(sysdate, 'yyyy-mm-dd-hh:mm' )||'-part-'|| batch_file_number ||'.sql';
       batch_file_number := batch_file_number + 1;
       fHandle := UTL_FILE.FOPEN('temp_dir',file_name , 'w');
       DBMS_OUTPUT.put_line('file: ' ||file_name );
       DBMS_OUTPUT.put_line('total_row_fetched: ' || total_row_fetched);
       
    end if;
    total_row_fetched := total_row_fetched + 1;
    -- write to file
    UTL_FILE.PUT_LINE(fHandle, out_columns||rtrim(out_values,',')||');');
 

    END LOOP;

    DBMS_SQL.close_cursor (curid);

  END LOOP;
  UTL_FILE.FCLOSE(fHandle);
  DBMS_OUTPUT.put_line('FINISHED');
  EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Exception: SQLCODE=' || SQLCODE || '  SQLERRM=' || SQLERRM);
    RAISE;
END;
/
 
 
```
