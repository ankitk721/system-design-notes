### Postgres Isolation levels
Postgres uses snapshots to provide isolations.

#### Read committed
Don't read dirty rows.

#### Repeatable read
My query, if repeated in the transaction, should return same results. Postgres gives consistent snapshot for reads but doesn't provide predicate-lock (i.e. it doesn't track your `select` condition and hence something might get inserted in the middle of your transation) which causes it to be at risk for Write-Skew/ Phantom.

```
-- Transaction A
BEGIN ISOLATION LEVEL REPEATABLE READ;

SELECT COUNT(*) FROM users WHERE email = 'ankit@example.com';
-- returns 0

-- Meanwhile, Transaction B
BEGIN;
INSERT INTO users(email) VALUES ('ankit@example.com');
COMMIT;

-- Back in Transaction A:
INSERT INTO users(email) VALUES ('ankit@example.com');
-- ERROR: duplicate key violation

```

Or
```
-- Transaction A
BEGIN REPEATABLE READ;
SELECT * FROM schedule WHERE doctor = 'Alice';  -- sees Alice is on
SELECT * FROM schedule WHERE doctor = 'Bob';    -- sees Bob is on
-- A thinks it's safe for Alice to go off-duty

-- Transaction B (parallel)
BEGIN REPEATABLE READ;
SELECT * FROM schedule WHERE doctor = 'Alice';  -- sees Alice is on
SELECT * FROM schedule WHERE doctor = 'Bob';    -- sees Bob is on
-- B thinks it's safe for Bob to go off-duty

-- A: UPDATE schedule SET on_duty = false WHERE doctor = 'Alice';
-- B: UPDATE schedule SET on_duty = false WHERE doctor = 'Bob';
-- COMMIT both

Oops! Now **both are off-duty**, violating your rule. This is called **write skew**, and it's a form of phantom anomaly.

```
#### Serializable

PostgreSQL’s **Serializable** isolation level (SSI) is specifically designed to **detect and prevent** this kind of issue. It will:

- Track the **predicate reads** (e.g., who looked at which data),
- Detect if **some transaction’s decision might have been invalidated by another’s write**, and
- **Abort one of the transactions** to preserve correctness.

##### How does Postgres protect against this in Serializable?
It tracks predicate level conditions that `T1 read balance being less than 100` and `T2 updated the balance`. Both wouldnt commmit in *serialized* world, so it aborts them. It maintains these dependencies of which transaction read what and is counting on what kind of predicate etc. When it detects a cycle, like the doctor example above, where both writes would impact each other's read, it aborts one of them. 

Note: this is all doable easily because Postgres is a single node database. And that Postgres doesn't provide serializable solution for distributed/sharded hosting.

##### How is Serializable transaction property ensured for transacion spawning multiple nodes?

Spanner uses a transaction co-ordinator and when committing a transaction T it keeps track of every key that it depended on and at the commit time, it verifies that none of those keys got their updates/state-change and if they did, it aborts this transaction.

