# uwd_take_home

- turnout_data has 205 duplicated StateFileIDs. Used Pandas to clean data before transferring into database

- environment_thermometer column in survey_data has a "<NA>" value, unlike any other null values (typically just "NA")
could have set column type in database to be varchar instead of integer, then updated values that matched "<NA>" to
  "NA", but issue is also fixed by reading data to a pandas df, then re_exporting, which is faster and maintains type
  integrity with the data warehouse, so opted for python script to do just that