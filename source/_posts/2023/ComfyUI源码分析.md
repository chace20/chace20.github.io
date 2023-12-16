---
title: ComfyUI源码分析
date: 2023-12-16 16:06:46
tags: 
 - AIGC
---

最近看到越来越多AI绘图圈的人开始使用ComfyUI，所以我也上手ComfyUI玩了一下SD Video，效果还不错。

![工作流](./img/2023/comfyui/comfyui_workflow.png)

![生成的视频](./img/2023/comfyui/comfyui_video.webp)

废话不多说，直接步入正题，我们来分析一下ComfyUI的源码。

# 启动过程

ComfyUI通过 `python main.py` 启动，所以代码入口就在 `main.py`。主要完成了以下的工作。

```python
# main.py

# 初始化server
server = server.PromptServer(loop)
# 初始化队列
q = execution.PromptQueue(server)
# 加载自定义节点
init_custom_nodes()
# 启动worker消费队列中的任务
threading.Thread(target=prompt_worker, daemon=True, args=(q, server,)).start()
```

# 运行workflow

在前端运行一次workflow，打开F12可以发现请求了后端的 `POST /prompt` 接口。所以 `server.py` 的 `POST /prompt` 接口就是运行workflow的入口。

server.py中定义了post_prompt，在482行将任务放入队列中。

```python
# server.py
# 482
self.prompt_queue.put((number, prompt_id, prompt, extra_data, outputs_to_execute))
```

**那么这个任务又是从哪里取出来的呢？在前面分析启动过程的时候，注意到同时启动了一个worker来消费任务队列中的任务。**

顺着代码继续，可以看在worker中取出队列中的任务，筛选出需要执行的节点，调用 `recursive_execute` 递归执行节点。

```python
# execution.py

# 150  recursive_execute
input_data_all = get_input_data(inputs, class_def, unique_id, outputs, prompt, extra_data)
# 实例节点类对象
obj = class_def()
output_data, output_ui = get_output_data(obj, input_data_all)
# 在get_output_data中调用map_node_over_list，最后真正地执行节点中的函数
results.append(getattr(obj, func)(**input_data_all))
```

另外在整个执行过程中也可以看到大量调用server.send_sync发送消息，这些消息会通过websocket发送给前端，前端接收小心来更新和展示进度。

```python
server.send_sync("executed", { "node": unique_id, "output": output_ui, "prompt_id": prompt_id }, server.client_id)
```

# 节点定义

ComfyUI的节点在 `nodes.py` 和 `comfy_extras` 目录下，具体的逻辑都在节点中完成。可以自己编写节点，放到 `custom_nodes` 目录下。

现在选择一个最初始的节点进行分析：

```python
class CheckpointLoaderSimple:
    @classmethod
    def INPUT_TYPES(s):
        return {"required": { "ckpt_name": (folder_paths.get_filename_list("checkpoints"), ),
                             }}
    RETURN_TYPES = ("MODEL", "CLIP", "VAE")
    FUNCTION = "load_checkpoint"

    CATEGORY = "loaders"

    def load_checkpoint(self, ckpt_name, output_vae=True, output_clip=True):
        ckpt_path = folder_paths.get_full_path("checkpoints", ckpt_name)
        out = comfy.sd.load_checkpoint_guess_config(ckpt_path, output_vae=True, output_clip=True, embedding_directory=folder_paths.get_folder_paths("embeddings"))
        return out[:3]
```

变量名都是字面意思，很容易理解：

```python
INPUT_TYPES: 输入参数
RETURN_TYPES: 输出参数
FUNCTION: 这个节点中的主函数
CATEGORY: 节点分组
```

在 `custom_nodes/example_node.py.example` 中有更完整的字段说明，如果要自己编写节点，照着写即可。

# 总结

看了一圈下来，明显感觉ComfyUI的代码架构比Automatic111的webui简单不少。虽然每个节点的逻辑实现还是会比较复杂，但是这种复杂性已经封装在节点中了，外层感受不到。而如果想要基于ComfyUI实现自己的业务逻辑，编写自定义节点即可。

总的来说，目前SD的UI最火的就是WebUI和ComfyUI了，这2者各有特色。**WebUI上手容易，而且有先发优势，借助插件机制，已经开拓了一个AI绘图生态；而ComfyUI虽然起步较晚，但是因为其灵活性，白盒化生图过程，对于高阶用户来说更容易定制，渐渐有追上webui的趋势。**

同时因为ComfyUI的灵活性，对于SD新特性的支持更快，比如SDXL、LCM、SDXL Turbo和SD Video，ComfyUI都是率先支持（据说ComfyUI的作者是Stability的员工）。