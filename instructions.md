# Task 1: SQL Exercise
Your coworker started collecting the data for this project, but didn’t finish it, and confesses to having done a careless job with it. It is your responsibility to make sure the data is complete, correct, and ready for upload into a table into Civis, our cloud-based data warehouse.

In 2020, we ran a program to track our effectiveness on voter persuasion. First, we conducted an SMS persuasion program on likely voters, in which we sent text messages to a targeted universe of voters with a message about candidate Jane Smith. We hoped the text messages would both persuade voters to support candidate Jane Smith over their opponents, and mobilize voters to cast a ballot in their state’s 2020 general election.  

Later, we conducted a phone survey polling some of those same voters on which candidate they supported in the upcoming election. 

After the election, we consulted publicly-available state voter files to measure whether the targeted voters actually voted on Election Day. 

We have attached the relevant files of the data. There are four files:

Universe.csv = the base file of the full targeted voter universe (targeted list of voters for outreach)

Turnout_data.csv = the records of who voted (a “1” on turnout_2020)

Survey_data.csv = the data returned from the phone vendor who completed the survey (a “1” on support_smith = yes, they support them)

Text_message_data.csv = the data returned from our texting platform

To submit your exercise, please send back two files: 

Data for upload into Civis - you can refer to the Upload Template. The data needs to be compiled into one single csv file with the relevant fields only. The file should allow us to track the following for all of the voters in the original universe (all of universe.csv records)
- Who was sent a text message
- Who was surveyed in the phone calls
- Who said they supported our candidate (a “1” on support_smith)
- Who ended up voting 

A short report in a spreadsheet format. Your manager needs the numbers from this data, so you also need to create a short report. Your manager likes clean and easy to read reports, so keep formatting and style in mind. In a spreadsheet, include counts for:
- Number of individuals who were sent a text message
- Support rate among those we surveyed
- Turnout rate among the universe
- Race counts of the targeted universe
- Age (bucketed ages 18 - 30, 31 - 45, 46 - 60, 61+) counts of the targeted universe
- Gender counts of the targeted universe
 

# Task 2: Explanation Email 
You receive an email from a coworker asking the following question: 

“According to the model, my friend Jamie has a support score of 50 (out of 100), but I’ve known Jamie for 20 years, and she is always in support of immigration issues. There must be something wrong with your model if it says she only has a 50% chance of supporting immigration issues.” 

How would you respond to your coworker? Assume that she is a smart, educated person with extensive political experience and little or no background in statistics. 
