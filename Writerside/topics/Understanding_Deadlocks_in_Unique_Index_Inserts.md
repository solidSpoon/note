# 三个线程插入同一行导致死锁

Hey! I recently learned something pretty cool about how databases handle certain situations, specifically when it comes to inserting records in tables with unique indexes. Let me explain it in a simple way.

Imagine you have a table, and it has a unique rule, like a primary key, which means no two rows can have the same value in that column. If you try to insert a value that's already there, the database has to be careful to keep things in order. It does this by locking that particular spot, sort of like putting a "do not enter" sign on a door.

Now, picture three people, or transactions, all trying to put the same number into this table. The first person gets there and locks the spot because they were first. When the second and third people try, they have to wait because the spot is already locked.

Here's where things get interesting. The second and third people can look at the spot but can't change anything until they lock it for themselves. It's like they need to upgrade their access from just looking around to actually making changes. But since they both need exclusive access to make the change, they end up waiting for each other, which creates a deadlock.

The database is smart, though. It notices this deadlock and decides to step in and resolve it by rolling back one of the transactions. It's like a referee stepping in to keep the game moving.

In a nutshell, deadlocks happen because each transaction is waiting to get exclusive access to insert the same record, causing them to block each other. The database then resolves this by rolling back one of the transactions to keep things moving smoothly.

Interesting, right?