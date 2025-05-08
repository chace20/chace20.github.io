---
title: Airflow源码分析(2)-xcom部分
date: 2019-09-01 12:07:22
tags: 
 - Airflow
 - 任务编排调度
---

## xcom简介
XComs(cross-communication)使得任务之间可以交换信息，允许更细粒度的控制和状态共享。XComs包含key, value, timestamp, 同时也包含创建xcom的任务实例的task_id, dag_id, execution_date等。

Task可以在运行时通过`xcom_push(key, value)`发送任意**可序列化成JSON**的对象。（其实支持pickle，但是已经被废弃）另外，task.execute()的返回值会默认发送到xcom，key为return_value。

Task中也可以通过`xcom_pull(task_id(s), key)`获取到一个或多个task的xcom值。不局限于下游。

详细介绍参考：[xcom](https://airflow.apache.org/concepts.html#xcoms)

## 详细分析

### 调用方式
在任务执行前，会先生成运行时的上下文context，然后调用task.execute(context)。因此在Operator的execute(context)方法中可以通过context['ti']得到当前的TaskInstance，然后调用`xcom_push`和`xcom_pull`

这里的context包含很多字段，比如ti, task, dag对象，还有Jinja渲染需要的字段。

### Context的字段定义
``` json
{
            'dag': task.dag,
            'ds': ds,
            'next_ds': next_ds,
            'next_ds_nodash': next_ds_nodash,
            'prev_ds': prev_ds,
            'prev_ds_nodash': prev_ds_nodash,
            'ds_nodash': ds_nodash,
            'ts': ts,
            'ts_nodash': ts_nodash,
            'ts_nodash_with_tz': ts_nodash_with_tz,
            'yesterday_ds': yesterday_ds,
            'yesterday_ds_nodash': yesterday_ds_nodash,
            'tomorrow_ds': tomorrow_ds,
            'tomorrow_ds_nodash': tomorrow_ds_nodash,
            'END_DATE': ds,
            'end_date': ds,
            'dag_run': dag_run,
            'run_id': run_id,
            'execution_date': self.execution_date,
            'prev_execution_date': prev_execution_date,
            'prev_execution_date_success': lazy_object_proxy.Proxy(
                lambda: self.previous_execution_date_success),
            'prev_start_date_success': lazy_object_proxy.Proxy(lambda: self.previous_start_date_success),
            'next_execution_date': next_execution_date,
            'latest_date': ds,
            'macros': macros,
            'params': params,
            'tables': tables,
            'task': task,
            'task_instance': self,
            'ti': self,
            'task_instance_key_str': ti_key_str,
            'conf': configuration,
            'test_mode': self.test_mode,
            'var': {
                'value': VariableAccessor(), # NOTICE
                'json': VariableJsonAccessor()
            },
            'inlets': task.inlets,
            'outlets': task.outlets,
        }
```

### xcom.py
Xcom是Model定义类，在里面实现了set(), get_one(), get_many(), delete()方法，对应数据库的增删查。

### taskinstance.py
上面说的xcom_pull()和xcom_push在BaseOperator和TaskInstance中均有定义。BaseOperator中只是简单调用了TaskInstance中的方法。

**TaskInstance部分代码：**
``` Python
    def xcom_push(
            self,
            key,
            value,
            execution_date=None):
        # 这里日期设置为未来一个时间，到时才可以被其他task发现
        """
        :param execution_date: if provided, the XCom will not be visible until
            this date. This can be used, for example, to send a message to a
            task on a future date without it being immediately visible.
        :type execution_date: datetime
        """

        if execution_date and execution_date < self.execution_date:
            raise ValueError(
                'execution_date can not be in the past (current '
                'execution_date is {}; received {})'.format(
                    self.execution_date, execution_date))

        XCom.set(
            key=key,
            value=value,
            task_id=self.task_id,
            dag_id=self.dag_id,
            execution_date=execution_date or self.execution_date)

    def xcom_pull(
            self,
            task_ids=None,
            dag_id=None,
            key=XCOM_RETURN_KEY,
            include_prior_dates=False):
        # 这里是为了得到那些设置为未来可见的xcom对象
        """
        :param include_prior_dates: If False, only XComs from the current
            execution_date are returned. If True, XComs from previous dates
            are returned as well.
        :type include_prior_dates: bool
        """

        if dag_id is None:
            dag_id = self.dag_id

        pull_fn = functools.partial(
            XCom.get_one,
            execution_date=self.execution_date,
            key=key,
            dag_id=dag_id,
            include_prior_dates=include_prior_dates)

        if is_container(task_ids):
            return tuple(pull_fn(task_id=t) for t in task_ids)
        else:
            return pull_fn(task_id=task_ids)
```
