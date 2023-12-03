---
title: Airflow源码分析(1)-executor部分
date: 2019-09-01 12:07:07
tags: 
 - Airflow
 - 任务编排调度
---

## Executor简介
Executor是在scheduler和worker之间的一个组件，主要作用是接收scheduler发过来的可执行task，然后根据自身类型决定task的运行环境。

目前有四种类型：

1. SequentialExecutor：Dag在单进程中顺序执行，用于测试跟开发
2. LocalExecutor：Dag在本地多进程执行，也是用于测试跟开发
3. CeleryExecutor：通过Celery下发任务到分布式集群。
4. DaskExecutor：下发任务到Dask集群上执行。Dask不支持队列。

## 流程分析
scheduler经过各种验证以后，终于将task标记为queued状态。

接下来，scheduler调用executor.queue_command将task_instance交给实际的executor。

``` python
self.executor.queue_command(simple_task_instance, command, priority=priority, queue=queue)
```


### BaseExecutor
我们打开`base_executor.py`，这个文件中`BaseExecutor`类作为具体executor的基类，可以看出executor大概的一个流程。

``` python
# 两个任务队列和一个事件buffer都是dict类型的，key是task_id
        self.queued_tasks = OrderedDict()
        self.running = {}
        self.event_buffer = {}

# 这个方法就是上面scheduler调用的方法，可以看出，BaseExecutor里维护了queued_tasks的任务队列，在这个方法里将新的task加入到了queued_tasks中
    def queue_command(self, simple_task_instance, command, priority=1, queue=None):
        key = simple_task_instance.key
        if key not in self.queued_tasks and key not in self.running:
            self.log.info("Adding to queue: %s", command)
            self.queued_tasks[key] = (command, priority, queue, simple_task_instance)
        else:
            self.log.info("could not queue task %s", key)

# scheduler会在自身的心跳间隔中重复调用executor.heartbeat()
# heartbeat()方法中，除了会打印executor的状态外，还会调用trigger_task()触发任务和sync()来同步状态
    def heartbeat(self):
        # Triggering new jobs
        if not self.parallelism:
            open_slots = len(self.queued_tasks)
        else:
            open_slots = self.parallelism - len(self.running)

        num_running_tasks = len(self.running)
        num_queued_tasks = len(self.queued_tasks)

        self.log.debug("%s running task instances", num_running_tasks)
        self.log.debug("%s in queue", num_queued_tasks)
        self.log.debug("%s open slots", open_slots)

        Stats.gauge('executor.open_slots', open_slots)
        Stats.gauge('executor.queued_tasks', num_queued_tasks)
        Stats.gauge('executor.running_tasks', num_running_tasks)

        self.trigger_tasks(open_slots)

        # Calling child class sync method
        self.log.debug("Calling the %s sync method", self.__class__)
        self.sync()

# 在trigger_tasks()方法中，最终会调用execute_async()方法来执行命令
    def trigger_tasks(self, open_slots):
        """
        Trigger tasks

        :param open_slots: Number of open slots
        :return:
        """
        sorted_queue = sorted(
            [(k, v) for k, v in self.queued_tasks.items()],
            key=lambda x: x[1][1],
            reverse=True)
        for i in range(min((open_slots, len(self.queued_tasks)))):
            key, (command, _, queue, simple_ti) = sorted_queue.pop(0)
            self.queued_tasks.pop(key)
            self.running[key] = command
            self.execute_async(key=key,
                               command=command,
                               queue=queue,
                               executor_config=simple_ti.executor_config)

# sync()和execute_async()都是抽象方法，但是从注释中我们可以看出其作用
# sync()用于收集状态，execute_async()是异步执行命令
    def sync(self):
        """
        Sync will get called periodically by the heartbeat method.
        Executors should override this to perform gather statuses.
        """
        pass

    def execute_async(self,
                      key,
                      command,
                      queue=None,
                      executor_config=None):  # pragma: no cover
        """
        This method will execute the command asynchronously.
        """
        raise NotImplementedError()

# change_state(...)只会改变event_buffer中的事件状态，但不会真正改变task的状态。
    def change_state(self, key, state):
        self.log.debug("Changing state: %s", key)
        self.running.pop(key, None)
        self.event_buffer[key] = state

```

总结一下executor的功能：
1. 接收来自scheduler的task，加入到自身维护的queued_tasks中
2. 在接收到scheduler的心跳后，打印自身的一些状态，在trigger_tasks(...)中将task从queued_tasks转移到running中，并最终调用execute_async(...)异步执行命令并调用sync(...)收集状态
3. scheduler可以调用get_event_buffer(...)获取executor的事件。executor改变自身维护的queued_tasks和running队列中task的状态时，都会上报事件到event_buffer中，从而可以被scheduler获取到

### SequentialExecutor
接下来以SequentialExecutor为例，看下execute_async和sync具体是怎么实现的

``` python
    def execute_async(self, key, command, queue=None, executor_config=None):
        self.commands_to_run.append((key, command,))

    def sync(self):
        for key, command in self.commands_to_run:
            self.log.info("Executing command: %s", command)

            try:
                # 在这里启动了一个子进程来执行task对应的command
                subprocess.check_call(command, close_fds=True)
                # 阻塞等待子进程返回，然偶上报success或者failed的状态
                self.change_state(key, State.SUCCESS)
            except subprocess.CalledProcessError as e:
                self.change_state(key, State.FAILED)
                self.log.error("Failed to execute task %s.", str(e))

        self.commands_to_run = []
```

**那么，task的调用到这里就结束了吗？**

一个`subprocess.check_call(command, close_fds=True)`就完了？task本身的状态是在哪改变的？对于HttpOperator，这个command又是如何执行的？

眉头一皱，发现事情并没有那么简单。

这里的command具体是什么呢？通过日志我们可以看到，其实是调用airflow CLI的run命令。
```
[2019-07-04 15:42:25,046] {base_executor.py:59} INFO - Adding to queue: ['airflow', 'run', 'example_json', 'echo_env', '2019-07-04T07:42:24.824155+00:00', '--local']
```

### cli.run
继续来看cli.run()

``` python
# run()中加载了配置文件，获取dag并实例化了TaskInstance，最终调用了_run()方法
@cli_utils.action_logging
def run(args, dag=None):
    # ...
    task = dag.get_task(task_id=args.task_id)
    ti = TaskInstance(task, args.execution_date)
    ti.refresh_from_db()

    ti.init_run_context(raw=args.raw)

    hostname = get_hostname()
    log.info("Running %s on host %s", ti, hostname)

    if args.interactive:
        _run(args, dag, ti)
    else:
        with redirect_stdout(ti.log, logging.INFO), redirect_stderr(ti.log, logging.WARN):
            _run(args, dag, ti)
    logging.shutdown()

# _run()方法中会根据参数来选择合适的，根据之前的参数'--local'，我们会进入到LocalTaskJob中去
def _run(args, dag, ti):
    if args.local:
        run_job = jobs.LocalTaskJob(
            task_instance=ti,
            mark_success=args.mark_success,
            pickle_id=args.pickle,
            ignore_all_deps=args.ignore_all_dependencies,
            ignore_depends_on_past=args.ignore_depends_on_past,
            ignore_task_deps=args.ignore_dependencies,
            ignore_ti_state=args.force,
            pool=args.pool)
        run_job.run()
    elif args.raw:
        ti._run_raw_task(
            mark_success=args.mark_success,
            job_id=args.job_id,
            pool=args.pool,
        )
    else:
        pickle_id = None
        if args.ship_dag:
        # ...
```

接下来是`base_job.py`，
``` python
    def run(self):
        Stats.incr(self.__class__.__name__.lower() + '_start', 1, 1)
        # Adding an entry in the DB
        with create_session() as session:
            self.state = State.RUNNING
            session.add(self)
            session.commit()
            id_ = self.id
            make_transient(self)
            self.id = id_

            try:
                self._execute()
                # In case of max runs or max duration
                self.state = State.SUCCESS
            except SystemExit:
                # In case of ^C or SIGTERM
                self.state = State.SUCCESS
            except Exception:
                self.state = State.FAILED
                raise
            finally:
                self.end_date = timezone.utcnow()
                session.merge(self)
                session.commit()

        Stats.incr(self.__class__.__name__.lower() + '_end', 1, 1)

    def _execute(self):
        raise NotImplementedError("This method needs to be overridden")

# LocalTaskJob中有具体的实现，可以看到是调用了一个TaskRunner
    def _execute(self):
        self.task_runner = get_task_runner(self)
        # 省略...
        self.task_runner.start()

# 接下来跳转到StandardTaskRunner
class StandardTaskRunner(BaseTaskRunner):
    """
    Runs the raw Airflow task by invoking through the Bash shell.
    """
    # ...
    def start(self):
        self.process = self.run_command()
    # ...

# 然后是run_command的具体实现
    def run_command(self, run_with=None, join_args=False):
        """
        Run the task command.

        :param run_with: list of tokens to run the task command with e.g. ``['bash', '-c']``
        :type run_with: list
        :param join_args: whether to concatenate the list of command tokens e.g. ``['airflow', 'run']`` vs
            ``['airflow run']``
        :param join_args: bool
        :return: the process that was run
        :rtype: subprocess.Popen
        """
        run_with = run_with or []
        cmd = [" ".join(self._command)] if join_args else self._command
        full_cmd = run_with + cmd

        self.log.info('Running: %s', full_cmd)
        proc = subprocess.Popen(
            full_cmd,
            stdout=subprocess.PIPE,
            stderr=subprocess.STDOUT,
            universal_newlines=True,
            close_fds=True,
            env=os.environ.copy(),
            preexec_fn=os.setsid
        )

        # Start daemon thread to read subprocess logging output
        log_reader = threading.Thread(
            target=self._read_task_logs,
            args=(proc.stdout,),
        )
        log_reader.daemon = True
        log_reader.start()
        return proc
```
代码看到这里发现又是subprocess.Popen(cmd)，那么这个时候的cmd内容是什么呢？可以从日志中看到。
```
[2019-07-04 13:15:38,406] {base_task_runner.py:133} INFO - Running: ['airflow', 'run', 'example_json', 'echo_env', '2019-07-04T05:15:36.239140+00:00', '--job_id', '120', '--raw', '--cfg_path', '/tmp/tmpg2123epz']
```

会发现又是airflow run，但是这个时候的cmd参数更多了，而且有一个`--raw`的参数。

### TaskInstance._run_raw_task

回到`_run(...)`，这个时候再执行命令会去另一个分支。于是会执行：
``` python
# cli.py
    elif args.raw:
        ti._run_raw_task(
            mark_success=args.mark_success,
            job_id=args.job_id,
            pool=args.pool,
        )

# taskinstance.py
    def _run_raw_task(
            self,
            mark_success=False,
            test_mode=False,
            job_id=None,
            pool=None,
            session=None):
        """
        Immediately runs the task (without checking or changing db state
        before execution) and then sets the appropriate final state after
        completion and runs any post-execute callbacks. Meant to be called
        only after another function changes the state to running.
        """

        try:
        # ...

        # 这里的task_copy类型就是Operator，也就是我们再定义DAG的时候选择的具体操作，通过调用Operator.execute(...)真正执行了我们想要运行的操作
                task_copy.pre_execute(context=context)
        # ...
                    result = task_copy.execute(context=context)
        # ...
                task_copy.post_execute(context=context, result=result)
        # ...
            self.refresh_from_db(lock_for_update=True)
            self.state = State.SUCCESS
        except AirflowSkipException:
            self.refresh_from_db(lock_for_update=True)
            self.state = State.SKIPPED
        except AirflowRescheduleException as reschedule_exception:
            self.refresh_from_db()
            # 在_handle_reschedule(...)中会将需要reschedule的任务加入到task_reschedule表中，状态为up_for_reschedule，等待被再次调度
            self._handle_reschedule(actual_start_date, reschedule_exception, test_mode, context)
            return
        except AirflowException as e:
            self.refresh_from_db()
            # for case when task is marked as success/failed externally
            # current behavior doesn't hit the success callback
            if self.state in {State.SUCCESS, State.FAILED}:
                return
            else:
            # 在handle_failure(...)中，会根据重试次数等信息将task状态设为up_for_retry或者failed
                self.handle_failure(e, test_mode, context)
                raise
        except (Exception, KeyboardInterrupt) as e:
            self.handle_failure(e, test_mode, context)
            raise

        # Success callback
        try:
            if task.on_success_callback:
                task.on_success_callback(context)
        except Exception as e3:
            self.log.error("Failed when executing success callback")
            self.log.exception(e3)

        # Recording SUCCESS
        self.end_date = timezone.utcnow()
        self.set_duration()
        if not test_mode:
            session.add(Log(self.state, self))
            session.merge(self)
        session.commit()
```

到这里，从scheduler将某个task分发给executor开始，一直到task被真正地执行的流程就完成了。

这里只分析了SequentialExecutor，对于CeleryExecutor，只是通过CeleryExecutor将cmd分发到远程worker上面执行了，接下来的流程是一样。

最后总结一下task是如何在worker上运行起来的：
1. 在worker上执行`airflow run <dag_id> <task_id> <execution_date> --local`命令
2. 进入到_run函数，选择local分支执行
3. 在local分支中绑定一个`LocalTaskJob`，并选择一个`BaseTaskRunner`作为task的执行环境。目前实现有`StandardTaskRunner`和`CgroupTaskRunner`两种
4. 在TaskRunner中调用run_command方法在子进程中继续执行命令，这时候命令为`airflow run <dag_id> <task_id> <execution_date> --raw`
5. 再次进入_run函数，选择raw分支执行
6. 在raw分支中调用_run_raw_task()，最后真正执行task.execute()方法
