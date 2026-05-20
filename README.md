# verl Architecture Report

> A deep dive into the architecture of verl's RL post-training system, focusing on Ray runtime orchestration, hybrid workers, rollout inference services, and model execution abstractions.

中文版：这是一份关于 **verl RL 后训练系统架构** 的源码分析报告，重点解释 verl 如何通过 Hydra、Ray、ResourcePool、PlacementGroup、Hybrid Worker、Rollout Server 和统一模型执行引擎，把 RLHF / PPO / GRPO 后训练流程组织成一个分布式训练系统。

---

## Overview

`verl` is not just a training script for PPO or GRPO.  
It is closer to a distributed runtime system for RL post-training.

This report analyzes how verl organizes:

- configuration and task startup
- Ray resource allocation
- actor / rollout / reference / critic workers
- hybrid worker colocation
- rollout inference services
- reward computation
- weight synchronization
- checkpoint and runtime coordination

The goal of this report is to help readers understand verl from a system architecture perspective, rather than only from the algorithm side.

---

## Report

📄 **Full PDF report**

- [verl 架构报告](./report/verl_architecture_report.pdf)

---

## What This Report Covers

### 1. Configuration and Entry Point

How `ppo_trainer.yaml` and `main_ppo.py` work together with Hydra and Ray to start a training job.

Key topics:

- Hydra configuration composition
- `run_ppo(config)`
- Ray initialization
- `TaskRunner.run(config)`
- dataset / tokenizer / worker / resource preparation

---

### 2. Ray Resource Management

How verl maps logical training roles to physical GPU/NPU resources.

Key topics:

- `ResourcePoolManager`
- `RayResourcePool`
- Ray Placement Groups
- bundle-based GPU allocation
- worker group creation
- actor / rollout / critic / reward resource mapping

---

### 3. Worker Initialization

How `RayPPOTrainer.init_workers()` turns configuration objects into real distributed runtime components.

Key topics:

- actor-side hybrid workers
- critic worker creation
- reference policy deployment modes
- colocated worker class generation
- `RayWorkerGroup.spawn()`
- reward loop manager
- rollout server manager
- checkpoint manager

---

### 4. Unified Model Engine Worker

How `engine_workers.py` provides a unified execution abstraction over different training backends.

Key topics:

- `TrainingWorker`
- `ActorRolloutRefWorker`
- actor / ref / rollout role composition
- `train_mini_batch()`
- `train_batch()`
- `infer_batch()`
- loss function injection
- metrics aggregation
- MFU calculation
- checkpoint loading and saving

---

### 5. Rollout Inference Service

How verl decouples rollout generation from the training loop.

Key topics:

- `LLMServerManager`
- async rollout
- rollout server replicas
- OpenAI-compatible inference interface
- vLLM rollout backend
- sticky session and prefix cache
- rollout weight update

---

### 6. Weight Synchronization

How actor training weights are synchronized into rollout inference replicas.

Key topics:

- actor-to-rollout weight bridge
- LoRA / adapter synchronization
- checkpoint engine backend
- rollout cache restore
- sleep / wake of rollout replicas

---

## Key Insights

Some of the most important conclusions from this report:

1. **verl is fundamentally a distributed runtime system.**  
   PPO / GRPO is only one layer of the system. Much of the complexity comes from runtime orchestration.

2. **Actor, rollout, and reference policy are often organized as hybrid workers.**  
   This allows verl to colocate related roles and reduce unnecessary process and communication overhead.

3. **Rollout generation is service-oriented.**  
   Instead of directly calling `model.generate()` inside the trainer, verl uses rollout servers to support asynchronous generation and better inference throughput.

4. **`TrainingWorker` is a unified model execution shell.**  
   It does not belong only to actor or critic. It abstracts over multiple backend engines and supports training, inference, checkpointing, metric aggregation, and performance statistics.

5. **Weight synchronization is a first-class system problem.**  
   The actor model used for training and the rollout model used for inference must be carefully synchronized across different runtime states.

---

## Suggested Repository Structure

```text
verl-architecture-report/
├── README.md
├── report/
│   └── verl_architecture_report.pdf
├── images/
│   ├── system_overview.png
│   ├── resource_pool.png
│   ├── init_workers.png
│   ├── engine_worker.png
│   └── rollout_pipeline.png
└── LICENSE
```

---

## Recommended Reading Order

If you are new to verl, I recommend reading the report in this order:

1. Overall architecture overview
2. Configuration and startup flow
3. Ray resource allocation
4. Worker initialization
5. Unified model engine worker
6. Rollout inference service
7. Weight synchronization and checkpoint coordination

---

## Related Projects

- [verl](https://github.com/volcengine/verl)
- [Ray](https://github.com/ray-project/ray)
- [vLLM](https://github.com/vllm-project/vllm)
- [OpenRLHF](https://github.com/OpenRLHF/OpenRLHF)
- [Hugging Face TRL](https://github.com/huggingface/trl)

---

## License

This repository is intended for technical learning and sharing.

Recommended license:

- Documentation: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
- Code snippets, if any: [MIT License](https://opensource.org/license/mit/)

If this repository only contains the report and diagrams, `CC BY 4.0` is usually a good choice.

---

## Author

Created by [@wangjunjie](https://github.com/junjiewang253-ctrl)

If you find this report useful, feel free to star the repository or share it with others interested in RLHF systems, LLM post-training, and AI infrastructure.
