# HOW IT WORKS

There are multiple parts in Resque:

1. [Enqueue jobs](#enqueue-jobs)
2. [Start workers](#start-workers) (`QUEUE=default bundle exec rake resque:work`)
3. [Process jobs](#process-jobs) (dequeue)
4. [Hooks](#hooks)

### Enqueue jobs

We need to use the `Resque.enqueue` method passing a class and the arguments that the job needs to perform.

**Constraints**

- The class need to define the `queue` instance variable **if not defined it will raise an exception**
- The class need to define the method `perform` that accept arguments

**How enqueue works**

- From the class it extract the `queue`
- Execute `before_enqueue_hooks`
- Do not enqueue if hooks fail
- Create a [Job](#job) passing the queue, the class, and the arguments
- Execute `after_enqueue_hooks`


# Job

- Validates class and queue (NOTE: At the moment this is done in the Job class, I think we would benefit if we move the logic of validating outside the job itself, let say to a separte class `JobValidator`)

- Push to `redis` via `DataStore`
  - Add the queue to the `set` in redis (`sadd`)
  - Add to the list (`rpush`) with the key `resque:queue:{name_of_the_queue}` and the encode value of the class and the arguments `"{\"class\":\"Demo::Job\",\"args\":[{}]}"`
  (NOTE: Both encoding and decoding could be move to it's own class)

### Start workers

The most important class is `Worker`

Process to set up a worker:
- Pass the QUEUES to the initializers, as well at some more arguments: VERBOSE level, PRE_SHUTDOWN_TIMEOUT, TERM_TIMEOUT, TERM_CHILD, GRACEFUL_TERM, RUN_AT_EXIT_HOOKS all of this attributes are set via ENV variables
- `QUEUES` validation (NOTE: probably a good idea to have a dedicated class for QUEUE logic, maybe `QUEUES`)

- `prepare` phase:
Daemonizes the worker if ENV['BACKGROUND'] is set and writes the process id to ENV['PIDFILE'] if set. Should only be called once per worker.

- reconnects to `redis` to avoid having to share a connection with the parent process, it tries up to 3 times

- `work` phase:
  - **startup**
    - *signal_register*
      * `TERM:` Shutdown immediately, stop processing jobs.
      * `INT:` Shutdown immediately, stop processing jobs.
      * `QUIT:` Shutdown after the current job has finished processing.
      * `USR1:` Kill the forked child immediately, continue processing jobs.
      * `USR2:` Don't process any new jobs
      * `CONT:` Start processing jobs again after a USR2
    - *start_heartbeat*
      * removes heartbeat associated with worker from redis list (`hdel`)
      * start heartbeat for worker store in redis list (`hset`) being the key the worker and the vale the server time. Also, store all heartbeat in a separate thread and use `ConditionalVariable` to exit the loop
    - *prune_dead_workers*
      * check for existing of key `resque:pruning_dead_workers_in_progress`, if it exists it will not do anything, if the key does not exists it will create it and set an expriation of `60` by default
      * Will get all workers and all [heartbeats](#heartbeats) and check if any of the worker are stale by checking the time at the moment and the last time the heartbeat was updated for that worker, if the time is higher that the `prune_interval` it will unregister the worker.
        * If the worker we are looking at do not belongs to the queues we are listening it will not unregister it.
    - *Run `before_fisrt_fork` hooks*
    - *Register worker*
      * it will add the worker to the `resque:workers' set `(sadd)` and set the key  worker + `started` with the value of the current time `(set)`

  - **loop**
    - it will get out of the loop if the shutdown attribute from the worker is set to `true`, normally done at the signal handler
    - it will sleep for the amount of time passed in the in the interval argument, by default is `5`
    - it no job found from the queues it will sleep for the amount of the interval argument (NOTE: maybe we can improve the efficience of the process if we keep a copy of the job of each queue in memory)
    - it will get the first job from the queue list
    - it will set the job worker
    - depending ENV variable `FORK_PER_JOB` or the system being able to fork process, it will fork or not
    - it will perform the job
    - it will store stats of the performed job
    - this steps are repeated until something fails or some signal is sent to the process
  - **unregister worker**
    - after we exit the loop we unregister the worker from the system
    - if for what ever reason we are processing a job, we will fail that execution of the job
    - it will `signal` the heartbeat thread allowing the heartbeat loop to exit
    - and last action would be to store some stats
    - if any error is raise at this level it will be rescue and log with infomration






### Heartbeats

When starting the worker it will store in Redis the of a hash with the key `"#{hostname}:#{pid}:#{@queues.join(',')}"` of the worker and the value is the time.







