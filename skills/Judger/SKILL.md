# Judger Skill

## Purpose

无 LangGraph 的独立评测流水线。支持四种任务类型：

- **code** — 代码生成评测（human-eval / mbpp），计算 pass@k
- **text2sql** — SQL 生成评测，SQLite 执行校验
- **general_text** — 通用文本评测（One-Eval DataFlowEvalTool）
- **math** — 数学/AIME 评测（生成、答案提取和判分在 Docker 镜像内完成）

## How to Invoke

**唯一入口：`loopai.skills.Judger.run()`**

`DB_PATH` 和 `TASK_ID` 从环境变量自动获取：

```bash
DB_PATH=api/db/db.sqlite3 TASK_ID=<task_id> \
python -c "from loopai.skills.Judger import run; run()"
```

或通过 CLI：

```bash
DB_PATH=api/db/db.sqlite3 TASK_ID=<task_id> loopai-judger
```

math 可使用 helper 一次完成配置注入和 CLI 启动：

```bash
conda activate loopai
DB_PATH=api/db/api.db TASK_ID=aime26-eval \
python examples/scripts/run_math_judger.py \
  --config-path examples/config/math_bench.json
```

仅注入配置时使用 `--inject-only`；断点恢复和指定起始步骤分别使用
`--resume`、`--from-step evaluate_math`。

推荐使用 shell 入口（Configer 和 Judger 均通过 CLI）：

```bash
conda activate loopai
export DB_PATH=api/db/api.db
export TASK_ID=aime26-eval
bash examples/scripts/run_math_judger.sh examples/config/math_bench.json
```

该脚本先调用 `loopai-configer update-task` 写入 `judger` 配置，再调用
`loopai-judger` 启动评测。首次修改 `setup.py` 后需重新安装项目以生成
`loopai-configer` 命令；开发环境也可将脚本中的命令替换为
`python -m loopai.skills.Configer.cli` 和 `python -m loopai.skills.Judger.cli`。

## Configuration

配置通过 **Configer skill** 写入 `TaskModel.state`，分两部分：

### 全局字段（state["judger"] 顶层，所有 bench 共享）

| 字段 | 默认值 | 说明 |
|---|---|---|
| `eval_model_path` | 无 | 模型路径（必填） |
| `eval_temperature` | `0` | 采样温度，bench 可覆盖 |
| `eval_top_p` | `0.95` | Top-P 采样，bench 可覆盖 |
| `eval_max_tokens` | `16384` | 最大输出 token 数（含思考推理），bench 可覆盖 |
| `eval_enable_thinking` | 不设置 | 思考模式开关（None 跟随模型默认 / True 开 / False 关），bench 可覆盖 |
| `eval_batch_size` | `10` | 批处理大小，bench 可覆盖 |
| `eval_case_num` | `10` | 每问题样本数，bench 可覆盖 |
| `eval_vllm_tensor_parallel_size` | `1` | vLLM 张量并行数 |
| `eval_vllm_gpu_memory_utilization` | `0.9` | vLLM GPU 显存利用率 |
| `cuda_visible_devices` | `"0"` | 指定 GPU |
| `output_dir` | `"./outputs"` | 输出根目录 |
| `eval_model_name` | 空 | `/v1/models` 暴露的模型名；留空使用 `eval_model_path` |
| `eval_top_k` / `eval_min_p` | `-1` / `0` | math 请求采样参数 |

### Bench 配置（state["judger"]）

所有评测集通过 `benchlist` 和 `extra_benchlist` 列表配置。**格式必须是 JSON 数组**（`[{...},{...}]`），**不是** JSONL（每行一个对象）：

```json
[{"name":"gsm8k","task_type":"general_text","problem_path":"/data/gsm8k/test.jsonl","eval_type":"key2_qa"},{"name":"human_eval","task_type":"code","problem_path":"/data/humaneval.jsonl","case_num":10}]
```

```json
{
  "benchlist": [
    {
      "name": "gsm8k",
      "task_type": "general_text",
      "problem_path": "/data/gsm8k/test.jsonl",
      "eval_type": "key2_qa",
      "key_mapping": {}
    },
    {
      "name": "human_eval",
      "task_type": "code",
      "problem_path": "/data/humaneval.jsonl",
      "case_num": 10,
      "batch_size": 10,
      "format_type": ""
    },
    {
      "name": "bird_dev",
      "task_type": "text2sql",
      "problem_path": "/data/bird/dev.jsonl",
      "text2sql_dir": "/data/bird/dev_databases",
      "case_num": 10,
      "batch_size": 10
    },
    {
      "name": "aime26",
      "task_type": "math",
      "problem_path": "/data/aime26_test.jsonl",
      "case_num": 2
    }
  ],
  "extra_benchlist": []
}
```

**bench entry 字段：**

| 字段 | code | text2sql | general_text | math | 说明 |
|---|---|---|---|---|---|
| `name` | ✅ 必填 | ✅ 必填 | ✅ 必填 | ✅ 必填 | bench 标识 |
| `task_type` | ✅ 必填 | ✅ 必填 | ✅ 必填 | ✅ 必填 | `code` / `text2sql` / `general_text` / `math` |
| `problem_path` | ✅ 必填 | ✅ 必填 | ✅ 必填 | ✅ 必填 | 问题文件路径 |
| `case_num` | 可选 10 | 可选 10 | — | 可选 10 | 每问题样本数；math 同时作为 val_n |
| `batch_size` | 可选 10 | 可选 10 | — | — | 批处理大小，bench 设了覆盖全局 |
| `temperature` | 可选 | 可选 | 可选 | 可选 | 覆盖全局 `eval_temperature` |
| `top_p` | 可选 | 可选 | 可选 | 可选 | 覆盖全局 `eval_top_p` |
| `max_tokens` | 可选 | 可选 | 可选 | 可选 | 覆盖全局 `eval_max_tokens` |
| `enable_thinking` | 可选 | 可选 | 可选 | 可选 | 覆盖全局 `eval_enable_thinking`，`false` 强制关闭思考 |
| `format_type` | 可选 | — | — | `human-eval` / `mbpp`，不设走默认 |
| `text2sql_dir` | — | ✅ 必填 | — | SQLite 数据库目录 |
| `eval_type` | — | — | ✅ 必填 | `key2_qa` / `key1_text_score` 等 |
| `key_mapping` | — | — | 可选 | 字段映射，可自动推断 |

**Per-bench 可选覆盖（重要）：** `case_num` / `batch_size` / `temperature` / `top_p` / `max_tokens` / `enable_thinking` 这 6 个字段**既可在全局设置，也可在单个 bench 里设置**。bench 里设置了就覆盖全局值，没设置就回落全局默认——用于「某个评测集需要特殊生成参数」的场景（例如某个 code 评测集需要更低温度、或某个 text2sql 评测集要关闭思考模式）。

```json
{
  "benchlist": [
    {
      "name": "human_eval",
      "task_type": "code",
      "problem_path": "/data/humaneval.jsonl",
      "temperature": 0.3,
      "enable_thinking": false
    },
    {
      "name": "human_eval_high",
      "task_type": "code",
      "problem_path": "/data/humaneval.jsonl",
      "temperature": 0.8,
      "max_tokens": 4096
    }
  ]
}
```

**主/附加区别：**

| | 主任务 | 附加任务 |
|---|---|---|
| 执行顺序 | 先 | 后 |
| 失败策略 | 记录失败 + `_save_task_progress` + 退出 | 记录失败，继续 |

### 预填写流程

```
1. configer_get_task(schema="states", section="judger", task_id="<task_id>")
2. 将缺失字段告知用户，征得确认后写入
3. configer_update_task("judger", {"benchlist": [...], "eval_model_path": "..."}, task_id="<task_id>")
```

## Pipeline

每个 bench entry 独立跑一遍完整流水线：

```
对每个 bench:
  _apply_bench_to_state → 注入 bench 字段到 state["judger"]
  → 按 task_type 选流水线:
    code/text2sql: validate → kill_vllm → start_vllm → format_data → generate → evaluate → kill_vllm_cleanup → finish
    general_text:  validate → eval_general_text → finish
    math:          validate → kill_vllm → start_vllm → evaluate_math (Docker) → kill_vllm_cleanup → finish
  → 收集结果到 bench_result / extra_bench_result
```

## Output

### stdout（emit_success）

```json
{
  "ok": true,
  "data": {
    "bench_result": [
      {"bench_name": "gsm8k", "task_type": "general_text",
       "output_result_path": "...", "metrics": {"accuracy": 0.94}}
    ],
    "extra_bench_result": [
      {"bench_name": "human_eval", "task_type": "code",
       "output_result_path": "...", "metrics": {"pass@1": 0.85}}
    ],
    "metrics": {"gsm8k": {"accuracy": 0.94}, "human_eval": {"pass@1": 0.85}}
  }
}
```

### 目录结构

```
outputs/<task_id>/
├── judger/
│   └── <version_id>/
│       ├── gsm8k/                      ← bench_name 子目录
│       │   ├── text_eval_summary_*.json
│       │   └── gsm8k_*_steps/
│       ├── human_eval/
│       │   ├── human_eval_sample.jsonl
│       │   ├── human_eval_result.jsonl
│       │   └── log.txt
│       └── bird_dev/
└── judger.pkl
```

### Configer 持久化

`_save_task_progress` 写入 `state.judger.bench_result` 和 `state.judger.extra_bench_result`，Analyzer 从中读取。

`math` 分支的数学评测逻辑由固定名称 `math-eval-loopai` 镜像提供。Judger
运行 `evaluate_math` 步骤时会先检查本地镜像；不存在则自动从
`loopai/skills/Judger/docker/math_eval` 构建（使用 `--network host`），无需手工
构建。运行时数据集只读挂载到容器，结果目录挂载到 `/outputs`；容器通过 host
network 访问 Judger 启动的 8911 vLLM 服务，不挂载宿主机 Conda 环境。

数学数据集必须是本地 JSON、JSONL 或 Parquet 文件。评测器自动将以下字段别名
归一化为 `problem` 和 `answer`：问题支持 `problem/question/prompt/query/input`，
答案支持 `answer/target/final_answer/solution`，因此不需要传入 `dataset` 类型参数。

### CLI 参数覆盖

安装项目后可直接使用 `loopai-judger`。默认从数据库任务读取配置；命令行参数
仅覆盖本次运行，不修改数据库：

```bash
DB_PATH=/path/to/api/db.sqlite3 \
loopai-judger \
  --task-id math-aime26-20260911-192302-ebdf0ac3 \
  --model-path /path/to/model \
  --dataset-path /path/to/aime26_test.jsonl \
  --cuda-visible-devices 4 \
  --case-num 2 \
  --max-tokens 38912
```

如果传入 `--config-path`，则配置文件会先覆盖并保存到指定任务的数据库状态，
然后再执行评测：

```bash
DB_PATH=/path/to/api/db.sqlite3 \
loopai-judger \
  --task-id math-aime26-20260911-192302-ebdf0ac3 \
  --config-path examples/config/math_bench.json
```

配置文件支持 `.json`、`.yaml`、`.yml`，内容可使用 `judger` 或
`default_states.judger` 结构。`task_id` 可从命令行、环境变量或配置文件读取，
优先级依次为命令行、环境变量、配置文件。

支持的覆盖项包括 `--temperature`、`--top-p`、`--top-k`、`--min-p`、
`--presence-penalty`、`--batch-size`、`--tensor-parallel-size`、
`--gpu-memory-utilization`、`--enable-thinking`/`--no-thinking` 和
`--output-dir`。`--resume` 与 `--from-step` 仍用于断点控制。

## Error Handling

每个步骤 `emit_error(exc, stream_writer=writer)`：
- stdout 输出 `{"ok": false, ...}` 
- judger.pkl 写入 `status=failed`
- taskruntime 表标记失败

所有 error `recoverable=true`，Codex 可引导用户修复后重试。

## Environment Variables

| 变量 | 来源 | 默认值 |
|---|---|---|
| `DB_PATH` | 环境变量 | 必填 |
| `TASK_ID` | 环境变量 | 必填 |
| `OUTPUT_DIR` | 环境变量 | `./outputs` |
| `CUDA_VISIBLE_DEVICES` | 环境变量 | `"0"` |
