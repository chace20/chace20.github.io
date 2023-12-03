---
title: Airflow源码分析(3)-jinja渲染部分
date: 2019-09-01 12:07:37
tags: 
 - Airflow
 - 任务编排调度
---

## 简介
Jinja渲染在Airflow用于参数跟字段的渲染，这里做一个简单的实现分析。

## 分析
模板渲染的流程是在TaskInstance()._run_raw_task()中进行的。

### _run_raw_task
_run_raw_task大致的逻辑如下：
``` Python
def _run_raw_task(...):
...
	try:
		# 渲染模板主要在这两个函数里面
		context = self.get_template_context()
		self.render_templates()
		# 真正地执行
		task_copy.pre_execute(context=context)
		result = task_copy.execute(context=context)
		task_copy.post_execute(context=context, result=result)
	except AirflowSkipException:
	    self.state = State.SKIPPED
	except AirflowRescheduleException：
		self._handle_reschedule()
	except Error:
		# 会处理retry的情况以及on_failure_callback()
		self.handle_failure()
	# 执行成功时的回调函数
	if task.on_success_callback:
	                task.on_success_callback(context)
```

### get_template_context
get_template_context中实际就是获取各种值，从返回值我们可以看到那些字段是可以通过context['variable']拿到的。
``` Python
    def get_template_context(self, session=None):
    ...
		    # 这里会发现trigger_dag中依然可以通过传递conf的方式覆盖参数值。conf是dict或者序列化的dict。
			# 如果在airflow.cfg中，dag_run_conf_overrides_params=True的话
			# 在pre_execute之前，会有params.update(dag_run.conf)的操作
			
            if configuration.getboolean('core', 'dag_run_conf_overrides_params'):
	            self.overwrite_params_with_dag_run_conf(params=params, dag_run=dag_run)
            
            # 这个类也是对应{var.value.your_variable_name}这种方式的实现
            class VariableAccessor:
            """
            Wrapper around Variable. This way you can get variables in templates by using
            {var.value.your_variable_name}.
            """
            def __init__(self):
                self.var = None

            def __getattr__(self, item):
                self.var = Variable.get(item)
                return self.var

            def __repr__(self):
                return str(self.var)
            
            return {
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

### render_templates
render_templates方法：
``` Python
    def render_templates(self):
        task = self.task
        # 这里又调用了一次get_template_context，因为有dry_run的情况
        # dry_run中不会调用get_template_context
        jinja_context = self.get_template_context()
        if hasattr(self, 'task') and hasattr(self.task, 'dag'):
	        # 这里也是user_defined_macros的实际处理
	        # 比如user_defined_macros=dict(foo='bar')，那么可以使用{{ foo }}拿到
            if self.task.dag.user_defined_macros:
                jinja_context.update(
                    self.task.dag.user_defined_macros)

        rt = self.task.render_template  # shortcut to method
        # 这里可以发现Operator中定义的template_fields的作用，只有这里面的字段才会被渲染
        for attr in task.__class__.template_fields:
            content = getattr(task, attr)
            if content:
                # 真正的渲染逻辑在task.render_template中
                # 参数为(属性名，内容，上下文)
                rendered_content = rt(attr, content, jinja_context)
                # 修改task的属性值为渲染后的内容
                setattr(task, attr, rendered_content)
```

### BaseOperator中的渲染
接下来我们进入到BaseOperator中，来看一下到底是怎么渲染的。
``` Python
    def render_template(self, attr, content, context):
        """
        Renders a template either from a file or directly in a field, and returns
        the rendered result.
        """
        # 首先，初始化jinja对象
        jinja_env = self.get_template_env()

        exts = self.__class__.template_ext
        if (
                isinstance(content, six.string_types) and
                any([content.endswith(ext) for ext in exts])):
            # 对于在template_ext=[]中定义的文件后缀，直接去读取文件内容渲染
            # 比如bash_command='test.sh', template_ext=['.sh']，那么会直接去将test.sh中的内容渲染出来
            return jinja_env.get_template(content).render(**context)
        else:
	        # 这里继续调用自身方法
            return self.render_template_from_field(attr, content, context, jinja_env)

    def get_template_env(self):
	    # 这里会调用到DAG的get_template_env，将DAG级别的macro, filter, template_search_path加载进来
        return self.dag.get_template_env() \
            if hasattr(self, 'dag') \
            else jinja2.Environment(cache_size=0)

    def render_template_from_field(self, attr, content, context, jinja_env):
        """
        Renders a template from a field. If the field is a string, it will
        simply render the string and return the result. If it is a collection or
        nested set of collections, it will traverse the structure and render
        all elements in it. If the field has another type, it will return it as it is.
        """
        rt = self.render_template
        if isinstance(content, six.string_types):
            result = jinja_env.from_string(content).render(**context)
            # 对于list, tuple, dict类型的变量，会递归地获取字段值，最终还是`基于字符串的渲染`或者`直接返回其他类型`
        elif isinstance(content, (list, tuple)):
            result = [rt(attr, e, context) for e in content]
        elif isinstance(content, dict):
            result = {
                k: rt("{}[{}]".format(attr, k), v, context)
                for k, v in list(content.items())}
        else:
            result = content
        return result
```
至此，模板渲染就完成了。task的一些属性会被渲染后的content代替。

另外还有两个方法，是将前面说的**模板文件**的内容加载进来，替换掉原来的**文件名**。同时在最后面`prepare_template()`方法，可以在这一步对模板文件的内容进行修改。

### 在渲染前修改模板内容

`resolve_template_files()`方法只在bag_dag和web view中有调用，起到将渲染前内容加载出来的作用。
``` Python

    def resolve_template_files(self):
        # Getting the content of files for template_field / template_ext
        for attr in self.template_fields:
            content = getattr(self, attr)
            if content is None:
                continue
            elif isinstance(content, six.string_types) and \
                    any([content.endswith(ext) for ext in self.template_ext]):
                env = self.get_template_env()
                try:
                    setattr(self, attr, env.loader.get_source(env, content)[0])
                except Exception as e:
                    self.log.exception(e)
            elif isinstance(content, list):
                env = self.dag.get_template_env()
                for i in range(len(content)):
                    if isinstance(content[i], six.string_types) and \
                            any([content[i].endswith(ext) for ext in self.template_ext]):
                        try:
                            content[i] = env.loader.get_source(env, content[i])[0]
                        except Exception as e:
                            self.log.exception(e)
        self.prepare_template()

    def prepare_template(self):
        """
        Hook that is triggered after the templated fields get replaced
        by their content. If you need your operator to alter the
        content of the file before the template is rendered,
        it should override this method to do so.
        """
        pass
```
