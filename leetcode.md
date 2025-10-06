### Question
*Design an online judge to provide problem submission and leaderboard functions of competetion*


### High level solution
- The worker runs the container as a child process of the worker program using something like:
- Your worker thread/process blocks and waits for docker container to execute and then gets its status code along with output

### Dive deep
- How would you pick among server execution, docker container execution vs VM execution?
- How would you ensure protection against malicious code execution?
- How would you ensure <2 second run time execution
  - Container startup first time latency
  - We reuse docker containers: <is it safe?>. Docker container reuse latency, latency for mounting input volume
  - What do we need to mount and how much time will it take to mount it? 
    - Mount Input and Output for test cases. <insert command for mounting>
  - File system permissions? Read or write or both? How are you protecting access?
  - How will you share the output back to submission worker?

- How would you build protection for system resources CPU, Memory, FileSystem, Network etc?
    - fork bomb- user AppArmor security profile and PID limit 
    - time limit - `timeout` option in container
    - vCPU assignment - fixed vCPU assignemnt
    - seccomp, app armour
    - network: none
    - disk space
        - Mount a directory specific to the container so that container is not able to access other directories of the host
        - If required make some as read-only-mounts to prevent abusing (e.g. if problems are being read from a mount, you dont want any container to corrupt the folder)
        - Mount only what is needed `docker run -v /tmp/sub_123:/app` {sourcepath:container_path}
        - `tmpfs-size=100m` not allowed to have more than 100MBs per container
- How would you ensure fast UI rendering?
- How would you rate limit your system to protect it from DDoS?
  - At the frontend server which takes in submission, we check for user's submission history and throttle if more than threshold number in last 1 minute.
- Quantification of scaling
    - CPU split
      - For JAVA code, around 200 MB for Base JDK Image and accounting for 100MB overhead, we can have 300-500MB for per container would mean we can't have more than 16 concurrent executions per host. But from CPU point of view, we need to pick how much vCPU we will allocate to each task and 0.5 vCPU is a good number to start. Lower than that might end up creating timeouts for optimal code and higher than that could end up end up burning money. Its a benchmarking and tradeoff discussion.
      - This means that for a 8 core server, we can run around 16 tasks concurrently. And referring to container latency from above, each execution will take ~1-2 seconds, so we can have around 16 concurrent tasks per host => 10000 submission/second ~ 500 servers. We can employ queue to buffer the odd peaks to prevent failing our system.


### Appendix

#### Comparison of VM vs Docker

| Metric           | Docker (container)   | VM (e.g., KVM, EC2)       |
| ---------------- | -------------------- | ------------------------- |
| Spin-up time     | \~100–300ms          | \~5–60 seconds            |
| Overhead         | Low (process-level)  | High (full OS boot)       |
| Memory footprint | 10s–100s of MBs      | 500MB–2GB+                |
| Use case         | Fast, frequent tasks | Long-lived, complex tasks |


#### Docker container execution latency

| Stage                        | Time(approx)      |
| ---------------------------- | ------------------ |
| Pull image (only first time) | 2–10 seconds       |
| Start container              | 100–300 ms         |
| Mount code, run commands     | 100–500 ms         |
| Compile code                 | 200–1000 ms (Java) |
| Execute code                 | 100 ms to few sec  |
| Tear down container          | 100–200 ms         |



#### Docker command

```
docker run --rm -m 256m --cpus=0.5 openjdk:17 sh -c "javac Solution.java && java Solution"
```

- runs a new container
- `rm` => delete container after it exits
- `sh -c` => run a shell command inside container 

#### FAQ
- why is first image pull slowers?
  - docker needs to download multi 100 MB image from DockerHub and store it in FileSystem. For later execution the image is already in disk and can be quickly used in container. 
  - container only does a readonly mount of the image.