# SQL Exercise

- Turnout_data has 205 duplicated StateFileIDs. I utilized Pandas to de-duplicate the data before sending to a database

- The environment_thermometer and support_smith columns in survey_data have a couple of different values for null. To
  ensure that copying of data into a database would be smooth, I read the data into a pandas dataframe, then 
  converted the dtypes to integer for both columns, which set the null value to a consistent "<NA>", making it easy
  to re-export to a new csv used for uploading to the database.
  
- I set the home and cell phone to both be the number that was associated with a person's given StateFileID. Though
there are more elegant solutions to this problem (like just setting the associated number to home numbers, and only 
  populating cell phone if an outbound text to that number received a response), I opted for this shortcut given time
  constraints.
  
- In the universe file, there is a race type of '()())()', which I updated to 'not given' value via pandas. This could
be a data entry error, or that the person did not give their race, but I opted for the latter for a cleaner report at
  the end.

- Note that the python script will overwrite the entire Excel file, and any nice formatting on the report page will be
wiped out should the script be run again.
  
## Queries

The two main queries used to generate the civis upload file and the report (respectively) are:

### Civis Upload
```
SELECT u.StateFileID as "StateFileID", 
       u.first_name as "First Name", null as "Middle Name", u.last_name as "Last Name", 
       null as "Address", null as "City", null as "State", null as "Zip", 
       u.phone_number as "Home Phone", u.phone_number as "Cell Phone", null as "Date of Birth",
       CASE WHEN u.gender = 'female' THEN 'F'
            WHEN u.gender = 'male' THEN 'M'
            ELSE null END as "Gender (only available as binary)", 
       CASE WHEN text.phone_number is null THEN False
            ELSE True END as sent_text, 
       CASE WHEN s.disposition = 1 THEN True
            ELSE False END as surveyed_by_phone, 
       s.support_smith, turnout.turnout2020 as voted
FROM universe u
LEFT JOIN (SELECT DISTINCT phone_number
           FROM text_message_data 
           WHERE message_direction = 'outbound') text
ON u.phone_number = text.phone_number
LEFT JOIN (SELECT phone_number, disposition, support_smith
           FROM survey_data) s
ON u.phone_number = s.phone_number
LEFT JOIN turnout_data turnout
ON u.StateFileID = turnout.StateFileID
;
```

### Short Report
```
WITH aggregate_table_source as(
SELECT u.StateFileID as state_file_id, 
       u.phone_number as phone, u.race as race, u.gender,
       CASE WHEN u.age between 18 and 30 THEN '18-30'
            WHEN u.age between 31 and 45 THEN '31-45'
            WHEN u.age between 46 and 60 THEN '46-60'
            WHEN u.age >= 61 THEN '61+'
            ELSE null END as age_bucket,
       CASE WHEN text.phone_number is null THEN False
            ELSE True END as sent_text,
       s.disposition, s.support_smith, turnout.turnout2020 as voted
FROM universe u
LEFT JOIN (SELECT DISTINCT phone_number
           FROM text_message_data 
           WHERE message_direction = 'outbound') text
ON u.phone_number = text.phone_number
LEFT JOIN (SELECT phone_number, disposition, support_smith
           FROM survey_data) s
ON u.phone_number = s.phone_number
LEFT JOIN turnout_data turnout
ON u.StateFileID = turnout.StateFileID)
SELECT 'num_texted' as metric_name, COUNT(CASE WHEN sent_text THEN 1 END) as metric
FROM aggregate_table_source
UNION ALL
SELECT 'support_rate' as metric_name, ROUND(SUM(support_smith)*1.0 / SUM(disposition), 4) as metric
FROM aggregate_table_source
UNION ALL
SELECT 'turnout_rate' as metric_name, ROUND(SUM(voted)*1.0 / COUNT(*), 4) as metric
FROM aggregate_table_source
UNION ALL
SELECT race as metric_name, COUNT(*) as metric
FROM aggregate_table_source
GROUP BY 1
UNION ALL
SELECT age_bucket as metric_name, COUNT(*) as metric
FROM aggregate_table_source
GROUP BY 1
UNION ALL
SELECT gender as metric_name, COUNT(*) as metric
FROM aggregate_table_source
GROUP BY 1;
```

Should you like to review the full python/SQL code I used to generate all this, see the end of this PDF
  

# Explanation Email

Hi Coworker,

I can totally understand the confusion around why the model suggests your friend only has a support score of 50, despite
you knowing that they almost always are in support of immigration issues. And we're so glad they are! The model itself
is not so much predicting how supportive an individual person would be on immigration issues, but how likely
someone who is similar to your friend is supportive of immigration issues. For example, let's say we have
two people who both live in states that tend to vote Republican, are of a higher income bracket, have similar occupations,
and both of whom have voted in 7 out of the last 10 elections they could vote in. Our model would rate both of those
people similarly for a support score, given they have similar demographic information, but it's totally possible that
one of these folks is always in support of immigration issues, and the other never is. On average though, people with
these particular details have a 50/50 chance of being supportive or not supportive of immigration issues. So while we
can't account for all the variety that exists for people who have similar details, the average value for folks like that
is still useful in creating our targeting universes.

Let me know if that's not clear, happy to dive into the details a little further so it all makes sense. Have a great rest
of your week!


# Full Code for Exercise 1
```python
import pandas as pd
import numpy as np

import psycopg2
import csv

# put all csv files into pandas dataframes for investigation and data manipulation
survey_df = pd.read_csv('survey_data.csv')
text_df = pd.read_csv('text_message_data.csv')
turnout_df = pd.read_csv('turnout_data.csv')
universe_df = pd.read_csv('universe.csv')

# convert the dtypes of support_smith and environment_thermometer, re-export to new file
survey_df['support_smith'] = survey_df['support_smith'].convert_dtypes()
survey_df['environment_thermometer'] = survey_df['environment_thermometer'].convert_dtypes()
survey_df.to_csv('survey_data_nanfix.csv', index=False, na_rep='NA')

# de-dup turnout and send to a new file
turnout_df.drop_duplicates(inplace=True)
turnout_df.to_csv('turnout_data_dedup.csv', index=False)

# fix the mysterious race type to 'not given' and send to a new file
universe_df['race'] = universe_df['race'].apply(lambda x: 'not given' if x == '()())()' else x)
universe_df.to_csv('universe_racefix.csv', index=False)

# a dictionary to associate tables with new csv files
table_names = {'survey_data': 'survey_data_nanfix', 
               'text_message_data': 'text_message_data', 
               'turnout_data': 'turnout_data_dedup', 
               'universe': 'universe_racefix'}

# Connect to an existing database
conn = psycopg2.connect(host="localhost",
                        database="uwd",
                        user="postgres",
                        password="postgres")

# Open a cursor to perform database operations
cur = conn.cursor()

table_creation = (
    """
    DROP TABLE IF EXISTS universe;
    CREATE TABLE universe (
        StateFileID VARCHAR(255) PRIMARY KEY,
        gender VARCHAR(255),
        last_name VARCHAR(255),
        phone_number BIGINT,
        race VARCHAR(255),
        age INTEGER,
        marital_status VARCHAR(255),
        first_name VARCHAR(255));
        """, 
    """
    DROP TABLE IF EXISTS turnout_data;
    CREATE TABLE turnout_data (
        StateFileID VARCHAR(255) PRIMARY KEY,
        turnout2020 INTEGER);    
        """,
    """
    DROP TABLE IF EXISTS text_message_data;
    CREATE TABLE text_message_data (
        phone_number BIGINT,
        message_direction VARCHAR(255),
        message_text VARCHAR(255));
        """,
    """
    DROP TABLE IF EXISTS survey_data;
    CREATE TABLE survey_data (
        phone_number BIGINT,
        attempted BOOLEAN,
        attempts INTEGER,
        disposition INTEGER,
        support_smith INTEGER,
        environment_thermometer INTEGER);
        """
)

# create table one by one
for command in table_creation:
    cur.execute(command)
# close communication with the PostgreSQL database server
cur.close()
# commit the changes
conn.commit()

# copy data from csv files to database tables
for table in table_names.keys():
    cur = conn.cursor()
    copy_sql = f"""
           COPY {table} FROM stdin WITH 
           CSV HEADER
           DELIMITER as ','
           NULL as 'NA'
           """
    with open(f"{table_names[table]}.csv", 'r') as f:
        cur.copy_expert(sql=copy_sql, file=f)
        conn.commit()
        cur.close()

# civis data upload query
cur = conn.cursor()

query = """
        SELECT u.StateFileID as "StateFileID", 
               u.first_name as "First Name", null as "Middle Name", u.last_name as "Last Name", 
               null as "Address", null as "City", null as "State", null as "Zip", 
               u.phone_number as "Home Phone", u.phone_number as "Cell Phone", null as "Date of Birth",
               CASE WHEN u.gender = 'female' THEN 'F'
                    WHEN u.gender = 'male' THEN 'M'
                    ELSE null END as "Gender (only available as binary)", 
               CASE WHEN text.phone_number is null THEN False
                    ELSE True END as sent_text, 
               CASE WHEN s.disposition = 1 THEN True
                    ELSE False END as surveyed_by_phone, 
               s.support_smith, turnout.turnout2020 as voted
        FROM universe u
        LEFT JOIN (SELECT DISTINCT phone_number
                   FROM text_message_data 
                   WHERE message_direction = 'outbound') text
        ON u.phone_number = text.phone_number
        LEFT JOIN (SELECT phone_number, disposition, support_smith
                   FROM survey_data) s
        ON u.phone_number = s.phone_number
        LEFT JOIN turnout_data turnout
        ON u.StateFileID = turnout.StateFileID
        ;
        """

cur.execute(query)
rows = cur.fetchall()
colnames = [desc[0] for desc in cur.description]

output_df = pd.DataFrame(rows, columns=colnames)
output_df.to_csv('civis_upload.csv', index=False)

cur.close()

# short report query
cur = conn.cursor()

query = """
        WITH aggregate_table_source as(
        SELECT u.StateFileID as state_file_id, 
               u.phone_number as phone, u.race as race, u.gender,
               CASE WHEN u.age between 18 and 30 THEN '18-30'
                    WHEN u.age between 31 and 45 THEN '31-45'
                    WHEN u.age between 46 and 60 THEN '46-60'
                    WHEN u.age >= 61 THEN '61+'
                    ELSE null END as age_bucket,
               CASE WHEN text.phone_number is null THEN False
                    ELSE True END as sent_text,
               s.disposition, s.support_smith, turnout.turnout2020 as voted
        FROM universe u
        LEFT JOIN (SELECT DISTINCT phone_number
                   FROM text_message_data 
                   WHERE message_direction = 'outbound') text
        ON u.phone_number = text.phone_number
        LEFT JOIN (SELECT phone_number, disposition, support_smith
                   FROM survey_data) s
        ON u.phone_number = s.phone_number
        LEFT JOIN turnout_data turnout
        ON u.StateFileID = turnout.StateFileID)
        
        SELECT 'num_texted' as metric_name, COUNT(CASE WHEN sent_text THEN 1 END) as metric
        FROM aggregate_table_source
        UNION ALL
        SELECT 'support_rate' as metric_name, ROUND(SUM(support_smith)*1.0 / SUM(disposition), 4) as metric
        FROM aggregate_table_source
        UNION ALL
        SELECT 'turnout_rate' as metric_name, ROUND(SUM(voted)*1.0 / COUNT(*), 4) as metric
        FROM aggregate_table_source
        UNION ALL
        SELECT race as metric_name, COUNT(*) as metric
        FROM aggregate_table_source
        GROUP BY 1
        UNION ALL
        SELECT age_bucket as metric_name, COUNT(*) as metric
        FROM aggregate_table_source
        GROUP BY 1
        UNION ALL
        SELECT gender as metric_name, COUNT(*) as metric
        FROM aggregate_table_source
        GROUP BY 1;
        """

cur.execute(query)
rows = cur.fetchall()
colnames = [desc[0] for desc in cur.description]

report_df = pd.DataFrame(rows, columns=colnames)
report_df['metric'] = report_df['metric'].astype(float)

report_df.to_excel('universe_report.xlsx', sheet_name='SQLoutput', index=False)

cur.close()

# close communication with the PostgreSQL database server
conn.close()
```
