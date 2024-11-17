---
author: chace
comments: true
date: 2024-11-17 12:55:07+00:00
layout: page
link: http://chace.in/whoami/
slug: whoami
title: whoami
---

Hello, I’m chace。

软件工程师，现居广州。

从事过2年的DevOps平台开发工作，对云原生、k8s、支撑大规模业务上云有丰富的经验。

近2年从事AIGC工作，负责AI绘图平台、机器学习平台以及AI for coding的落地，持续探索AI的能力边界。

欢迎交流学习～

# 联系方式

- Email:  chao9420@gmail.com
- 知乎:  https://www.zhihu.com/people/chace20
- Github:  https://github.com/chace20

# 工作经历

2024-至今    AI for coding平台

1. 静态代码扫描：集成SonarQube、CodeQL、PHPStan等多种扫描工具，支持Python、Java、C++、Go、PHP、JS等主流语言的静态代码扫描
2. C++扫描在UE上的适配：攻克C++扫描在大型游戏项目上的落地难题，支持3000W+行代码扫描，扫描效果达到业务方预期
3. AI Review：基于dify搭建评测管线，构造测试用例，评测不同AI Prompt和模型的能力在AI Review上的能力。

2023-2024    AI生图平台的开发和维护

1. 对 webui、comfyui 进行二次开发，拆分webui的单体架构使之能分布式运行，支持300+ 卡A30和4090 GPU资源调度
2. 工程架构优化：重构后端架构，采用长短队列分离、共享独占算力池、模型缓存等多种手段，满足100QPS 出图吞吐，模型缓存命中率≥80%
3. GPU推理优化：集成onediff/blade等推理优化能力，建立benchmark评估不同推理框架、不同显卡在绘图场景下的性能
4. API解决方案：作为平台输出SaaS能力，为游戏内嵌玩法、营销玩法提供API支持。开发自助接入平台，实现一键部署API到公有云的能力，单API交付成本从7.5人缩短至0.5人天
5. AI Infra平台维护：掌握KubeFlow + KServe框架上的二次开发，实现基于混合云架构的算力池和云上缓存加速
6. 外部资料：[网易游戏机器学习云平台助力AI应用落地实践](https://developer.aliyun.com/ebook/8109/112701)

2020-2022   DevOps平台的开发和维护

1. 从0到1：调研Helm/ArgoCD/Kustomize/KubeVela等多种编排工具，最终自研并设计API层-编排层-执行层的3层架构，同时满足游戏、Web应用、Istio应用的编排需求
2. 性能优化：单体调度性能优化，拆分项目独占调度资源，共享资源。支持100+项目，3W+ pod的管理
3. 业务出海：支撑游戏出海，对接AWS、GCP，实现国内海外的统一管理
4. 稳定性建设：与QA团队配合在组内落地单测流水线，覆盖率从40%提升到77%；落地线上巡检用例30+，提前发现线上故障
5. 其他：多次组织k8s分享沙龙，培养一名新人进组并转正
6. 外部资料：[效率提升10倍，网易游戏面向终态的应用交付实践](https://www.51cto.com/article/708517.html)

# 教育经历

**西安电子科技大学**

计算机技术，硕士

经历：搭建和维护实验室的云平台，支持PhDs跑实验以及其他在线服务的运行

**电子科技大学**

计算机科学与技术，本科

经历：大三担任[SysLab](https://github.com/Sys-Lab)工作室学生负责人，负责招新培训、组织参加比赛、创新创业项目的孵化等

# 专业认证

**[AWS高级架构师认证](https://ouvhhkkplk.feishu.cn/file/Qco8bwGJNo6tVzxYfKxc6XeFnQX)**

熟悉AWS的网络、容器平台ECS/EKS等内容

**[CKA(Certified Kubernetes Administrator)认证](https://ouvhhkkplk.feishu.cn/file/C8nqb0dHloGL1NxFw5Mc0qotnph)**

具备生产级别K8S部署、维护能力
