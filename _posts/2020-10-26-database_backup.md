---
layout: post
title: Database Backup
---

<p class="message">
  This challange asked me to analyze a database backup and answer various questions.
  I was unable to solve all the questions but the ones I did are described below. 
</p>

**Q1**\
The database backup contained multiple .bson and metadata.json files.\
In order to more easily read the .bson file I converted them to json using bsondump install from [MongoDB packages](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-debian/)\
Then using awk to select the correct feild and using wc -l to count the returned lines I found the number of users in the database.

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/log_analysis/databasebackup_q1_command1.png">
<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/log_analysis/databasebackup_q1_command2.png">
<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/log_analysis/databasebackup_q1_proof.png">

**Q2**\
When looking at the original users file I know the admin feild is set to 1 when the user is admin.\
After searching for this with no results I reexamined the users file and found that some users were having the admin feild set as a double instead of an int.\
Knowing this I rewrote my search and found the admin user.

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/log_analysis/databasebackup_q2_command.png">
<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/log_analysis/databasebackup_q2_proof.png">

**Q3**\
Knowing the syntax for guest usernames from analysing the file in previous questions allowed me to grep and count all guest users.

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/log_analysis/databasebackup_q3_command.png">
<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/log_analysis/databasebackup_q3_proof.png">

**Q4**\
First I used grep on the file to return all users with a friends feild.
Then I used grep again to remove all users that had an empty friends list and counted the number of users using wc -l.

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/log_analysis/databasebackup_q4_command.png">
<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/log_analysis/databasebackup_q4_proof.png">

**Q5**\
Copying a password from a user and running it through a hash analyser reveals the hashing algorthim to be md5.

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/log_analysis/databasebackup_q5_command.png">
<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/log_analysis/databasebackup_q5_proof.png">

**Q7**\
I used grep to find the number of users with the password feild set to "".

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/log_analysis/databasebackup_q7_command.png">
<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/log_analysis/databasebackup_q7_proof.png">

**Q8**\
First I searched for users who had the resetPasswordExpires in their user and then exluded all users whos value was set to "" as this meant they had not
reset their passwords.

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/log_analysis/databasebackup_q8_command.png">
<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/log_analysis/databasebackup_q8_proof.png">
