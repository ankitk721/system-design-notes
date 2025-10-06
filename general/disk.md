### Disk Generals

- WAL:  Write stuff you're planning to do so if you crash and recover, the HD will retain the info for process.

1. What is the main way of writing to disk?
Process uses `write or read()` system calls to add bytes from user-space to kernel buffer and doing `flush()` basically forces kernel-buffer data to sync to HD. `flush()` calls `fsync()` system call. Effectively this is the call flow
`write()` -> kernel buffer -> `flush()/fsync()` -> disk

2. What is the bottleneck of disk writs?

- It is the frequency and throughput you can maintain with `fsync()` system call. Typically for commodity HDD its upto 1000 tps which reall gets in the way of achieving high throughput if you're fsync-ing every request.

3. So what options do I have to make it faster?

- This is typcal tradeoff discussion for durability vs performance. If possible, plan to fsync every 5 ms. This might take your throughput to 5-10k+. Or use SSD which has higher throughput for `fsync()`

4. What if I can't live with risk of losing writes but want higher throughput?
- Use a quorum across machines with 5ms `fsync()` interval for each of them- so they'll keeping stuff in memory and will provide higher throughput. The only latency will be due to network latency, quorum will provide durability. DynamoDb uses something similar. 
