---
author: ["Ivan Kwong"]
title: 'Safe Update Operation in Postgresql Using Sqlalchemy'
date: 2021-05-08T01:51:19-04:00
description: "Avoid race conditions in PostgreSQL updates using SQLAlchemy techniques."
summary: "The article outlines three methods for safe update operations in PostgreSQL using SQLAlchemy to avoid race conditions: performing direct updates without pre-reading, using with_for_update to lock rows during transactions, and employing optimistic locking with a version column to manage concurrent updates. These techniques help maintain data integrity in multi-process environments."
tags: ["python", "postgres", "database"]
categories: ["python"]
showToc: true
TocOpen: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
editPost:
    URL: "https://github.com/ivankwongtszfung/my-tech-blog/tree/main/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---
## Problem Statement
If you are using the default isolation mode in PostgreSQL. When you try to update your database in your web application, ACID transaction will **not** safe you from race condition. if you run the code below with more than one process running. (which multi processes are very common for web servers)
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import Session
# setup
engine = create_engine(...)
session = Session(engine)
# read from the table Account
account = session.query(Account).get(1)
# modify the record, account is decimal
account.amount = account.amount + 100
session.commit()
```
Why is this not safe? This is because SELECT will not pose any locks in row level. It blocks only when update happens which it locks the row exclusively (`FOR UPDATE/FOR NO KEY UPDATE`). Race condition could happen in this scenario.

| Transaction 1 (T1)| Transaction 2 (T2)|
| ------------- |:-------------:|
| START | |
| SELECT amount | START  |
| amount += 100 | SELECT amount |
| UPDATE amount | amount += 100 |
| COMMIT | |
| | UPDATE amount |
| | COMMIT `SAME value as T1` |

Since T1 doesn't commit before T2 reads, so T2 will select the same value as T1 did. which eventually commits the same thing again.

So, how to prevent this situation in default isolation mode. (Read Committed mode)

## Solutions
### Solution 1: Just don't read
The above snippet is implementing with an anti-pattern called 
`read-modify-write`. One way to prevent that is to not read and change the value directly with the column value. Given read is not super necessary in this scenario.
```python
# same session setting as above
session.query(Account).filter_by(id=1)\
     .update({"amount": Account.amount + 100})
session.commit()
```
### Solution 2: Update Lock
There is some scenarios that you want to do a complicated modification and you just have to read first. In this case, you could use update lock when you read from the database. In SQLAlchemy, there is a method called `with_for_update` which locks the row you want to change with a `FOR UPDATE` lock. `FOR UPDATE` lock is not self compatible, another transactions have to wait until this transaction release the lock.
```python
# same session setting as above
# locks the row that id = 1
account = session.query(Account).filter_by(id=1)\
     .with_for_update().one()
account.amount = account.amount + 100
session.commit()  # save and release the lock
```
### Solution 3: Version tracking (optimistic locking)
If you want to rollback all other processes which run any updates on the same rows. You can add a version column in the table to track the updates on this row. Let say we have version v in some rows, when you wants to update the row, you will search the row along with version v and update the version to v + 1.
```SQL
update account set version = v + 1, ... where version = v ...;
```
This is quite annoying to implement ourselves, Fortunately, most of the ORM library support version tracking. In SQLAlchemy, you can set the version column name in the mapper, so when you update, it will manage the version for you and if some processes work on the same row, it will raise an Exception.
```python
class Account(Base):
    __tablename__ = "account"
    ...
    version = Column(Integer, nullable=False)

    __mapper_args__ = {"version_id_col": version}
```
```python
def version_tracking(change):
    try:
        account = session.query(Account).get(1)
        account.amount = account.amount + change
        print_account(account, change)
        session.commit()
    except StaleDataError:
        print("someone has changed the account, plz retry.")
        # some actions...

```

All the solutions given above, assume we are using the Read Committed mode which is the default isolation model in Postgres. I also assume you are not using any aggregation, when you select the target rows. There are some more ways to prevent the same problem, such as using a stricter isolation mode so that the transactions will act like working serially. However, we will not go through this in this article.
### For more Info.
[All Source Code](https://github.com/ivankwongtszfung/proof_of_concept/tree/main/psql_acid_test)  
[SQLAlchemy for update (kite)](https://www.kite.com/python/docs/sqlalchemy.sql.expression.GenerativeSelect.with_for_update)  
[SQLAlchemy Version Tracking](https://docs.sqlalchemy.org/en/14/orm/versioning.html)  
[PSQL Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)  
[PSQL Locking](https://www.postgresql.org/docs/current/explicit-locking.html)  


