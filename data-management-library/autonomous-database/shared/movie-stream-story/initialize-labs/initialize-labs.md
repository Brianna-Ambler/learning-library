# Initializing Labs

## Introduction
This page contains the script that can be used to create and load the tables required for the labs.  

The workshop has been designed to run from beginning to end.  However, you may want to skip certain labs - which means that you may have also skipped some prerequisites.  You can run the initialization script to ensure that your database environment is in the proper state.  The script will drop and recreate tables required to run the labs.  These are the same tables that are defined in the data loading lab.

Estimated Time: 5 minutes

## **Initialization Script** 

Go to SQL Worksheet in SQL Tools and login as the **moviestream** user.  Then, copy and paste the script below into the SQL Worksheet and click **Run Script**.

The script will drop and recreate the MovieStream tables and load them from files in object storage.  In addition, primary keys, foreign keys and IS JSON constraints are defined.  This helps ensure data quality as well as more sophisticated analytics using JSON and graph queries.

 ```
<copy>
-- This is the gold image for creating the schema
define uri_landing = 'https://objectstorage.us-ashburn-1.oraclecloud.com/p/tLBDQf0prua1qyczMj321erjEGwCy1kJwi5Y3B0YWBnssNmfRxRhQFRbeS799RH1/n/adwc4pm/b/moviestream_landing/o'
define uri_gold = 'https://objectstorage.us-ashburn-1.oraclecloud.com/p/S-9E_eBGGoo9xNm3QP-DoNPr8xlmibT52mXsirQdvmiv_FfRTFmAx2ajUqFY5RCW/n/adwc4pm/b/moviestream_gold/o'
define csv_format = '{"dateformat":"YYYY-MM-DD", "skipheaders":"1", "delimiter":",", "ignoreblanklines":"true", "removequotes":"true", "blankasnull":"true", "trimspaces":"lrtrim", "truncatecol":"true", "ignoremissingcolumns":"true"}'
define pipe_format = '{"dateformat":"YYYY-MM-DD", "skipheaders":"1", "delimiter":"|", "ignoreblanklines":"true", "removequotes":"true", "blankasnull":"true", "trimspaces":"lrtrim", "truncatecol":"true", "ignoremissingcolumns":"true"}'
define json_format = '{"skipheaders":"0", "delimiter":"\n", "ignoreblanklines":"true"}'
define parquet_format = '{"type":"parquet",  "schema": "all"}'

-- initialize
drop table ext_genre;
drop table ext_movie;
drop table ext_customer_contact;
drop table ext_customer_extension;
drop table ext_customer_segment;
drop table ext_pizza_location;
drop table ext_custsales;
drop table custsales;
drop table customer_contact;
drop table customer_extension;
drop table customer;
drop table genre;
drop table movie;
drop table customer_segment;
drop table pizza_location;
drop table time;
drop view v_custsales;
 
 
-- Test:  query object storage with the credential
-- SELECT *
-- FROM DBMS_CLOUD.LIST_OBJECTS('OBJ_STORE_CRED', '&uri_gold/');
 
 
-- Create a time table  over 2 years.  Used to densify time series calculations
exec dbms_output.put_line(systimestamp || ' - create time table')

create table time as
select trunc (to_date('20210101','YYYYMMDD')-rownum) as day_id
from dual connect by rownum < 732;
comment on table time is 'This is a densified time dimension table with the following hierarchy day-week-month-quarter-year';

alter table time
add (
    day_name as (to_char(day_id, 'DAY')),
    day_dow as (to_char(day_id, 'D')),
    day_dom as (to_char(day_id, 'DD')),
    day_doy as (to_char(day_id, 'DDD')),
    week_wom as (to_char(day_id, 'W')),
    week_woy as (to_char(day_id, 'WW')),
    month_moy as (to_char(day_id, 'MM')),
    month_name as (to_char(day_id, 'MONTH')),
    month_aname as (to_char(day_id, 'MON')),
    quarter_name as ('Q'||to_char(day_id, 'Q')||'-'||to_char(day_id, 'YYYY')),
    quarter_qoy as (to_char(day_id, 'Q')),
    year_name as (to_char(day_id, 'YYYY'))
);
 
-- Using public buckets  so credentials are not required
-- Create external tables then do a CTAS

begin
    dbms_output.put_line(systimestamp || ' - create external tables');
    dbms_output.put_line(systimestamp || ' - ... ext_genre');
    dbms_cloud.create_external_table(
        table_name => 'ext_genre',
        file_uri_list => '&uri_gold/genre/genre.csv',
        format => '&csv_format',
        column_list => 'genre_id number, name varchar2(30)'
        );

    dbms_output.put_line(systimestamp || ' - ... ext_customer_segment');
    dbms_cloud.create_external_table(
        table_name => 'ext_customer_segment',
        file_uri_list => '&uri_landing/customer_segment/customer_segment.csv',
        format => '&csv_format',
        column_list => 'segment_id number, name varchar2(100), short_name varchar2(100)'
        );       

    dbms_output.put_line(systimestamp || ' - ... ext_movie'); 
    dbms_cloud.create_external_table(
        table_name => 'ext_movie',
        file_uri_list => '&uri_gold/movie/movies.json',
        format => '&json_format',
        column_list => 'doc varchar2(30000)'
        );
  
    dbms_output.put_line(systimestamp || ' - ... ext_custsales');
    dbms_cloud.create_external_table(
        table_name => 'ext_custsales',
        file_uri_list => '&uri_gold/custsales/*.parquet',
        format => '&parquet_format',
        column_list => 'MOVIE_ID NUMBER(20,0),
                        LIST_PRICE BINARY_DOUBLE,
                        DISCOUNT_TYPE VARCHAR2(4000 BYTE),
                        PAYMENT_METHOD VARCHAR2(4000 BYTE),
                        GENRE_ID NUMBER(20,0),
                        DISCOUNT_PERCENT BINARY_DOUBLE,
                        ACTUAL_PRICE BINARY_DOUBLE,
                        DEVICE VARCHAR2(4000 BYTE),
                        CUST_ID NUMBER(20,0),
                        OS VARCHAR2(4000 BYTE),
                        DAY_ID date,
                        APP VARCHAR2(4000 BYTE)'
    ); 
    
    dbms_output.put_line(systimestamp || ' - ... ext_pizza_location');       
    dbms_cloud.create_external_table(
        table_name => 'ext_pizza_location',
        file_uri_list => '&uri_landing/pizza_location/*.csv',
        format => '&csv_format',
        column_list => 'PIZZA_LOC_ID NUMBER,
                        LAT NUMBER,
                        LON NUMBER,
                        CHAIN_ID NUMBER,
                        CHAIN VARCHAR2(30 BYTE),
                        ADDRESS VARCHAR2(250 BYTE),
                        CITY VARCHAR2(250 BYTE),
                        STATE VARCHAR2(26 BYTE),
                        POSTAL_CODE VARCHAR2(38 BYTE),
                        COUNTY VARCHAR2(250 BYTE)'
        ); 
    
    dbms_output.put_line(systimestamp || ' - ... ext_customer_contact');  
    dbms_cloud.create_external_table(
        table_name => 'ext_customer_contact',
        file_uri_list => '&uri_gold/customer_contact/*.csv',
        format => '&csv_format',
        column_list => 'CUST_ID                  NUMBER,         
                        LAST_NAME                VARCHAR2(200 byte), 
                        FIRST_NAME               VARCHAR2(200 byte), 
                        EMAIL                    VARCHAR2(500 byte), 
                        STREET_ADDRESS           VARCHAR2(400 byte), 
                        POSTAL_CODE              VARCHAR2(10 byte), 
                        CITY                     VARCHAR2(100 byte), 
                        STATE_PROVINCE           VARCHAR2(100 byte), 
                        COUNTRY                  VARCHAR2(400 byte), 
                        COUNTRY_CODE             VARCHAR2(2 byte), 
                        CONTINENT                VARCHAR2(400 byte),
                        YRS_CUSTOMER             NUMBER, 
                        PROMOTION_RESPONSE       NUMBER,         
                        LOC_LAT                  NUMBER,         
                        LOC_LONG                 NUMBER' 

        ); 

    dbms_output.put_line(systimestamp || ' - ... ext_customer_extension');
    dbms_cloud.create_external_table(
        table_name => 'ext_customer_extension',
        file_uri_list => '&uri_landing/customer_extension/*.csv',
        format => '&csv_format',
        column_list => 'CUST_ID                      NUMBER,         
                        LAST_NAME                    VARCHAR2(200 byte),
                        FIRST_NAME                   VARCHAR2(200 byte), 
                        EMAIL                        VARCHAR2(500 byte),
                        AGE                          NUMBER,         
                        COMMUTE_DISTANCE             NUMBER,         
                        CREDIT_BALANCE               NUMBER,         
                        EDUCATION                    VARCHAR2(40 byte),
                        FULL_TIME                    VARCHAR2(40 byte),
                        GENDER                       VARCHAR2(20 byte), 
                        HOUSEHOLD_SIZE               NUMBER,         
                        INCOME                       NUMBER,         
                        INCOME_LEVEL                 VARCHAR2(20 byte), 
                        INSUFF_FUNDS_INCIDENTS       NUMBER,         
                        JOB_TYPE                     VARCHAR2(200 byte),
                        LATE_MORT_RENT_PMTS          NUMBER,         
                        MARITAL_STATUS               VARCHAR2(8 byte),
                        MORTGAGE_AMT                 NUMBER,         
                        NUM_CARS                     NUMBER,         
                        NUM_MORTGAGES                NUMBER,         
                        PET                          VARCHAR2(40 byte), 
                        RENT_OWN                     VARCHAR2(40 byte), 
                        SEGMENT_ID                   NUMBER,
                        WORK_EXPERIENCE              NUMBER,         
                        YRS_CURRENT_EMPLOYER         NUMBER,         
                        YRS_RESIDENCE                NUMBER'
        ); 

    dbms_output.put_line(systimestamp || ' - external tables created.') ;


    --  Create tables from external tables

    dbms_output.put_line(systimestamp || ' - create pizza_locations');
    execute immediate 'create table pizza_location as select * from ext_pizza_location';
 
    dbms_output.put_line(systimestamp || ' - create genre');
    execute immediate 'create table genre as select * from ext_genre';
 
    dbms_output.put_line(systimestamp || ' - create customer_segment');
    execute immediate 'create table customer_segment as select * from ext_customer_segment';
 
    dbms_output.put_line(systimestamp || ' - create customer_contact');
    execute immediate 'create table customer_contact as select * from ext_customer_contact';

    dbms_output.put_line(systimestamp || ' - create customer_extension');
    execute immediate 'create table customer_extension as select * from ext_customer_extension';

    dbms_output.put_line(systimestamp || ' - create movie');
    execute immediate 'create table movie as
        select
            cast(m.doc.movie_id as number) as movie_id,
            cast(m.doc.title as varchar2(200 byte)) as title,   
            cast(m.doc.budget as number) as budget,
            cast(m.doc.gross as number) gross,
            cast(m.doc.list_price as number) as list_price,
            cast(m.doc.genre as varchar2(4000)) as genres,
            cast(m.doc.sku as varchar2(30 byte)) as sku,   
            cast(m.doc.year as number) as year,
            to_date(m.doc.opening_date, ''YYYY-MM-DD'') as opening_date,
            cast(m.doc.views as number) as views,
            cast(m.doc.cast as varchar2(4000 byte)) as cast,
            cast(m.doc.crew as varchar2(4000 byte)) as crew,
            cast(m.doc.studio as varchar2(4000 byte)) as studio,
            cast(m.doc.main_subject as varchar2(4000 byte)) as main_subject,
            cast(m.doc.awards as varchar2(4000 byte)) as awards,
            cast(m.doc.nominations as varchar2(4000 byte)) as nominations,
            cast(m.doc.runtime as number) as runtime,
            substr(cast(m.doc.summary as varchar2(4000 byte)),1, 4000) as summary
        from ext_movie m';
 
    dbms_output.put_line(systimestamp || ' - create custsales');
    execute immediate 'create table custsales as select * from ext_custsales';

    -- Table combining the two independent ones
    dbms_output.put_line(systimestamp || ' - create combined customer');
    execute immediate 'create table CUSTOMER
            as
            select  cc.CUST_ID,                
                    cc.LAST_NAME,              
                    cc.FIRST_NAME,             
                    cc.EMAIL,                  
                    cc.STREET_ADDRESS,         
                    cc.POSTAL_CODE,            
                    cc.CITY,                   
                    cc.STATE_PROVINCE,         
                    cc.COUNTRY,                
                    cc.COUNTRY_CODE,           
                    cc.CONTINENT,              
                    cc.YRS_CUSTOMER,           
                    cc.PROMOTION_RESPONSE,     
                    cc.LOC_LAT,                
                    cc.LOC_LONG,               
                    ce.AGE,                    
                    ce.COMMUTE_DISTANCE,       
                    ce.CREDIT_BALANCE,         
                    ce.EDUCATION,              
                    ce.FULL_TIME,              
                    ce.GENDER,                 
                    ce.HOUSEHOLD_SIZE,         
                    ce.INCOME,                 
                    ce.INCOME_LEVEL,           
                    ce.INSUFF_FUNDS_INCIDENTS, 
                    ce.JOB_TYPE,               
                    ce.LATE_MORT_RENT_PMTS,    
                    ce.MARITAL_STATUS,         
                    ce.MORTGAGE_AMT,           
                    ce.NUM_CARS,               
                    ce.NUM_MORTGAGES,          
                    ce.PET,                    
                    ce.RENT_OWN,    
                    ce.SEGMENT_ID,           
                    ce.WORK_EXPERIENCE,        
                    ce.YRS_CURRENT_EMPLOYER,   
                    ce.YRS_RESIDENCE
            from CUSTOMER_CONTACT cc, CUSTOMER_EXTENSION ce
            where cc.cust_id = ce.cust_id';
 
    -- View combining data
    dbms_output.put_line(systimestamp || ' - create view v_custsales');
    execute immediate 'CREATE OR REPLACE VIEW v_custsales AS
            SELECT
                cs.day_id,
                c.cust_id,
                c.last_name,
                c.first_name,
                c.city,
                c.state_province,
                c.country,
                c.continent,
                c.age,
                c.commute_distance,
                c.credit_balance,
                c.education,
                c.full_time,
                c.gender,
                c.household_size,
                c.income,
                c.income_level,
                c.insuff_funds_incidents,
                c.job_type,
                c.late_mort_rent_pmts,
                c.marital_status,
                c.mortgage_amt,
                c.num_cars,
                c.num_mortgages,
                c.pet,
                c.promotion_response,
                c.rent_own,
                c.work_experience,
                c.yrs_current_employer,
                c.yrs_customer,
                c.yrs_residence,
                c.loc_lat,
                c.loc_long,   
                cs.app,
                cs.device,
                cs.os,
                cs.payment_method,
                cs.list_price,
                cs.discount_type,
                cs.discount_percent,
                cs.actual_price,
                1 as transactions,
                s.short_name as segment,
                g.name as genre,
                m.title,
                m.budget,
                m.gross,
                m.genres,
                m.sku,
                m.year,
                m.opening_date,
                m.cast,
                m.crew,
                m.studio,
                m.main_subject,
                nvl(json_value(m.awards,''$.size()''),0) awards,
                nvl(json_value(m.nominations,''$.size()''),0) nominations,
                m.runtime
            FROM
                genre g, customer c, custsales cs, customer_segment s, movie m
            WHERE
                cs.movie_id = m.movie_id
            AND  cs.genre_id = g.genre_id
            AND  cs.cust_id = c.cust_id
            AND  c.segment_id = s.segment_id';

    -- Add constraints and indexes
    dbms_output.put_line(systimestamp || ' - creating constraints and indexes');

    execute immediate 'alter table genre add constraint pk_genre_id primary key("GENRE_ID")';

    execute immediate 'alter table customer add constraint pk_customer_cust_id primary key("CUST_ID")';
    execute immediate 'alter table customer_extension add constraint pk_custextension_cust_id primary key("CUST_ID")';
    execute immediate 'alter table customer_contact add constraint pk_custcontact_cust_id primary key("CUST_ID")';
    execute immediate 'alter table customer_segment add constraint pk_custsegment_id primary key("SEGMENT_ID")';

    execute immediate 'alter table movie add constraint pk_movie_id primary key("MOVIE_ID")';
    execute immediate 'alter table movie add CONSTRAINT movie_cast_json CHECK (cast IS JSON)';
    execute immediate 'alter table movie add CONSTRAINT movie_genre_json CHECK (genres IS JSON)';
    execute immediate 'alter table movie add CONSTRAINT movie_crew_json CHECK (crew IS JSON)';
    execute immediate 'alter table movie add CONSTRAINT movie_studio_json CHECK (studio IS JSON)';
    execute immediate 'alter table movie add CONSTRAINT movie_awards_json CHECK (awards IS JSON)';
    execute immediate 'alter table movie add CONSTRAINT movie_nominations_json CHECK (nominations IS JSON)';
    
    execute immediate 'alter table pizza_location add constraint pk_pizza_loc_id primary key("PIZZA_LOC_ID")';

    execute immediate 'alter table time add constraint pk_day primary key("DAY_ID")';

    -- foreign keys
    execute immediate 'alter table custsales add constraint fk_custsales_movie_id foreign key("MOVIE_ID") references movie("MOVIE_ID")';
    execute immediate 'alter table custsales add constraint fk_custsales_cust_id foreign key("CUST_ID") references customer("CUST_ID")';
    execute immediate 'alter table custsales add constraint fk_custsales_day_id foreign key("DAY_ID") references time("DAY_ID")';
    execute immediate 'alter table custsales add constraint fk_custsales_genre_id foreign key("GENRE_ID") references genre("GENRE_ID")';
    dbms_output.put_line(systimestamp || 'Done.');
 end;
 /


</copy>
 ```


## Acknowledgements
* **Author** - Marty Gubar, Product Manager - Server Technoloties
* **Last Updated By/Date** - <Marty Gubar, Group, July 2021>
