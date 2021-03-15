## Objective
Understanding indexing

## Lab

### Setup

* Important: This lab will create a `lab5.db` with size around 600MB, make sure you have enough disk space.

* Prepare lab5 database
  ```
  > cp ../db/labs.db lab5.db
  ```

* Connect to a database
   ```
   > sqlite3 lab5.db
   ```

* Create a table: `big_cards`
  ```
  create table big_cards as select * from cards;
  insert into big_cards select * from cards;
  insert into big_cards select * from big_cards;
  insert into big_cards select * from big_cards;
  insert into big_cards select * from big_cards;
  insert into big_cards select * from big_cards;
  insert into big_cards select * from big_cards;
  insert into big_cards select * from big_cards;
  insert into big_cards select * from big_cards;
  insert into big_cards select * from big_cards;
  insert into big_cards select * from big_cards;
  ```

A data set for the collection of cards for [Hearthstone](https://playhearthstone.com/en-us/),
the popular online card game by Blizzard. This data set is freely available from 
[Kaggle](https://www.kaggle.com/jeradrose/hearthstone-cards).

### Indexing

Indexes are used to retrieve data from the database very fast. The users cannot see the indexes, they are just used to speed up searches/queries.

#### EXPLAIN QUERY PLAN 

In order to see the impact of our indexing efforts, we will use the `EXPLAIN QUERY PLAN` feature of SQLite.

First: turn SQL time on
```sql
sqlite> .timer ON
```

Example:

```sql
sqlite> EXPLAIN QUERY PLAN SELECT count(*) FROM big_cards;
```

Record output below:

```
QUERY PLAN
`--SCAN TABLE big_cards
Run Time: real 0.000 user 0.000037 sys 0.000011

AFTER CHANGES DESCRIBED IN FINDINGS:

QUERY PLAN
`--SCAN TABLE big_cards
2886656
Run Time: real 0.832 user 0.062769 sys 0.236526
```

#### Using Indexes to improve performance

As a developer you know that your application will perform the query below *a lot*. It's not quite as fast as you would like it to be so you start performance tuning it.

**Query**: `select card_id, name from big_cards where race = 'TOTEM';`

You suspect an index will help, but before you make any changes you want to get a baseline explain plan and execution time. You do this using `EXPLAIN QUERY PLAN`.

**Execute**: `EXPLAIN QUERY PLAN SELECT card_id, name FROM big_cards WHERE race = 'TOTEM';`

Record output below:

```
QUERY PLAN
`--SCAN TABLE big_cards
Run Time: real 0.000 user 0.000074 sys 0.000027

AFTER CHANGES DESCRIBED IN FINDINGS:

QUERY PLAN
`--SCAN TABLE big_cards
** Other output excluded **
Run Time: real 0.920 user 0.272728 sys 0.301368
```

You suspect that an index on the race column will help. Let's create it.

**Execute**: `CREATE INDEX IDX1_big_cards ON big_cards(race);`

**Execute**: `EXPLAIN QUERY PLAN SELECT card_id, name FROM big_cards WHERE race = 'TOTEM';`

Record output below:

```
QUERY PLAN
`--SEARCH TABLE big_cards USING INDEX IDX1_big_cards (race=?)
Run Time: real 0.000 user 0.000038 sys 0.000013

AFTER CHANGES DESCRIBED IN FINDINGS:

QUERY PLAN
`--SEARCH TABLE big_cards USING INDEX IDX1_big_cards (race=?)
** Other output excluded **
Run Time: real 1.386 user 0.029565 sys 0.068280
```

Would it be possible to satisfy the query with an index only and further speed up the query?

**Execute**: `CREATE INDEX IDX2_big_cards ON big_cards(race, card_id, name);`

**Execute**: `EXPLAIN QUERY PLAN SELECT card_id, name FROM big_cards WHERE race = 'TOTEM';`

Record output below:

```
QUERY PLAN
`--SEARCH TABLE big_cards USING COVERING INDEX IDX2_big_cards (race=?)
Run Time: real 0.001 user 0.000053 sys 0.000015

AFTER CHANGES DESCRIBED IN FINDINGS:

QUERY PLAN
`--SEARCH TABLE big_cards USING COVERING INDEX IDX2_big_cards (race=?)
** Other output excluded **
Run Time: real 0.023 user 0.006131 sys 0.015401
```

If you issue command `VACUUM big_cards;` and re-analyze you will likely see an explain plan that *is* satisfied by the index (and consequently much faster). However, subsequent updates to the table would cause this query to go back to the table to check the visibility map.

**Execute**: `VACUUM;`

**Execute**: `EXPLAIN QUERY PLAN SELECT card_id, name FROM big_cards WHERE race = 'TOTEM';`

Record output below:

```
QUERY PLAN
`--SEARCH TABLE big_cards USING COVERING INDEX IDX2_big_cards (race=?)
Run Time: real 0.039 user 0.000000 sys 0.037139

AFTER CHANGES DESCRIBED IN FINDINGS:

QUERY PLAN
`--SEARCH TABLE big_cards USING COVERING INDEX IDX2_big_cards (race=?)
** Other output excluded **
Run Time: real 0.055 user 0.005092 sys 0.046755
```

#### The performance cost of Indexes 

In general, we don't want to create unused indexes because they incur a performance penalty. The penalty is often minimal unless the application has a very high rate of updates. But it is something to be aware of.

Let's analyze an update of every row while our two indexes exist:

**Execute**: `EXPLAIN QUERY PLAN update big_cards set race = 'FOO';`

Note the Execution time.

Record output below:

```
QUERY PLAN
`--SCAN TABLE big_cards
Run Time: real 0.000 user 0.000066 sys 0.000036

AFTER CHANGES DESCRIBED IN FINDINGS:

QUERY PLAN
`--SCAN TABLE big_cards
Run Time: real 144.108 user 21.554400 sys 25.051749
```


Now let's drop the indexes and try again:

**Execute**: `drop index idx1_big_cards;`

**Execute**: `drop index idx2_big_cards;`

**Execute**: `EXPLAIN QUERY PLAN update big_cards set race = 'BAR';`

Record output below:

```
QUERY PLAN
`--SCAN TABLE big_cards
Run Time: real 0.000 user 0.000000 sys 0.000098

AFTER CHANGES DESCRIBED IN FINDINGS:

QUERY PLAN
`--SCAN TABLE big_cards
Run Time: real 8.520 user 1.973711 sys 2.243990
```

Does the update took less time without the indexes? 
Your answer:
```
From my findings, the update took significantly less time without the indexes, 
especially when I ran it the correct way after I made the changes described below.
```

Describe your findings of this Lab 5 from the recorded outputs, is everything working fine? or is anything not working? etc. Please indicate your SQLite version:

```
SQLite version: 3.26.0 2018-12-01 12:34:55 bf8c1b2b7a5960c282e543b9c293686dccff272512d08865f4600fb58238alt1
Findings: In this lab, I made it all the way through the queries and noticed that the queries 
were not being executed and outputting like they should be. 
When running the queries without the Explain Query Plan, I noticed that they took considerably 
longer to execute, and also produced the output that the user would want.
That is when I discovered that you can turn on the Explain Query Plan by using .eqp ON. 
Then, when you execute the normal statements without the Explain Query Plan prefix, 
the commands will run as normal and produce the output, but also still give the Explain Query Plan output. 
Therefore, as shown above in all of the recorded outputs, I included
both the times that I ran before the change, and also the ones after the change.


```

ps. Use this command to check your SQLite version. `sqlite3 --version`

The moral of this lab is: create the indexes you need, but *only* the indexes you need.

ps. Please remember to delete lab5.db and release disk space especially if you are using either ap1 or rlogin.

### Your turn
Now it's your turn to apply what you've learned.

1. Complete this `Lab_5.md` with your result 
2. Covert your `Lab_5.md` to a PDF file. You can use something like this: https://www.markdowntopdf.com/
3. Rename your filename to `yourPID_Lab_5.pdf` and submit it to Canvas `Lab 5 assignment`
