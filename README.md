# MySQL Internals Training Lab

Hands-on labs exploring MySQL internals — query execution lifecycle, optimizer behavior, index design, and InnoDB architecture. Built for backend and DevOps engineers who want to write better queries and stop guessing why MySQL is slow.

---

## Who This Is For

Engineers who already use MySQL day-to-day and want to move from intuition to a systematic understanding of what happens under the hood. Each lab is self-contained, hands-on, and focused on observable behavior over theory.

---

## Prerequisites

- MySQL 8.x instance with a user that has `CREATE`, `INSERT`, `SELECT`, `INDEX`, and `PROCESS` privileges
- Basic familiarity with SQL — `SELECT`, `WHERE`, `JOIN`, `GROUP BY`
- A MySQL client (`mysql` CLI, DBeaver, or similar)

---

## Sessions

| # | Topic | What You Will Learn |
|---|-------|---------------------|
| [01](./01-query-execution-lifecycle/) | Query Execution Lifecycle | What MySQL does between receiving a query and returning results — Parser, Optimizer, Executor, InnoDB |
| [02](./02-index-design/) | Index Design | How indexes are structured, when MySQL uses them, and how to design them around your query patterns |
| [03](./03-innodb-internals/) | InnoDB Internals | Clustered indexes, the buffer pool, redo/undo logs, and how InnoDB manages data on disk |
| [04](./04-query-optimization/) | Query Optimization | Reading EXPLAIN output, rewriting slow queries, and working with the optimizer rather than against it |
| [05](./05-transactions-and-locks/) | Transactions and Locks | ACID guarantees, InnoDB locking mechanics, deadlocks, and how to diagnose and avoid lock contention |

> **Recommended order:** work through sessions 01 → 05 sequentially. Each session builds on concepts introduced in the previous one.

---

## How Each Lab Is Structured

Every session README follows the same format:

- **Learning objectives** — what you will be able to do by the end
- **Setup** — any schema or data required
- **Exercises** — step-by-step with expected output and explanations
- **Questions** — prompts to check your understanding
- **Takeaways** — the one or two things worth remembering from each exercise
- **Quick reference** — commands and patterns to keep

---

## Running the Labs

Clone the repo and open the session README in your editor or GitHub, then run the SQL in your MySQL client alongside it.

```bash
git clone https://github.com/<your-org>/mysql-internals-training-lab.git
cd mysql-internals-training-lab
```

Each lab creates its own database so sessions do not interfere with each other.

---

## Contributing

Found a mistake or want to add a session? Open a PR. Keep the same structure as existing READMEs so the format stays consistent for participants.