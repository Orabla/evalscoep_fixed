# EvalScope 使用指南

EvalScope 是一个灵活的模型评测框架，支持多后端集成和多样化任务配置。以下是官方仓库的修复版本，修复了模型竞技场模式：

---

## 环境准备

### 1. 创建 Conda 环境
```bash
conda create -n evalscope python=3.10
conda activate evalscope
```

### 2. 安装 EvalScope
#### 基础安装
```bash
pip install -e . 
```


## 配置准备

### 1. 配置 OpenAI API 环境变量
```bash
export OPENAI_API_KEY="YOUR_API_KEY"
export OPENAI_BASE_URL="YOUR_API_BASE_URL"


```
确保/root/lm_eval/evalscope/evalscope/models/model.py下可以访问到这俩个变量

### 2. 下载评测模型
```bash
# 示例：下载 Qwen 系列模型
modelscope download --model Qwen/Qwen2.5-0.5B-Instruct --local_dir ./qwen2.5_0.5B_model
modelscope download --model Qwen/Qwen1.5-0.5B-Chat --local_dir ./qwen1.5_0.5B_model
modelscope download --model Qwen/Qwen2.5-7B-Instruct --local_dir ./qwen2.5_7B_model
```

### 3. 自定义配置文件
```bash
# 编辑评测配置文件
vim cfg_arena.yaml

# 配置示例：
question_file: /root/lm_eval/evalscope/my_eval_config/question.jsonl

# candidate models to be battled
answers_gen:
    Qwen2-3B-bistu-sft:
        model_id_or_path: /root/llm_model/qwen2.5_3B
        precision: torch.float16
        enable: true           # enable or disable this model      # TODO: tokenizer issue
        template_type: default-generation
        generation_config:
            do_sample: true
            max_new_tokens: 256
            top_k: 20
            top_p: 0.75
            temperature: 0.3
        output_file: registry/data/arena/answers/answer_Qwen2-3B.jsonl
    Qwen2-0.5B:
        model_id_or_path: /root/llm_model/qwen2.5_0.5B_model
        # revision: v1.1.8       # revision of model, default is NULL
        precision: torch.float16
        enable: true           # enable or disable this model      # TODO: tokenizer issue
        template_type: default-generation
        generation_config:
            do_sample: true
            max_new_tokens: 256
            top_k: 20
            top_p: 0.75
            temperature: 0.3
        output_file: registry/data/arena/answers/answer_Qwen2-0.5B.jsonl

# Auto-reviewer(GPT-4) config
reviews_gen:
    enable: true
    reviewer:
        # class reference of auto reviewer(GPT-4)
        ref: evalscope.evaluator.reviewer.auto_reviewer:AutoReviewerGpt4
        args:
            max_tokens: 1024
            temperature: 0.2
            # options: pairwise, pairwise_baseline, single (default is pairwise)
            mode: pairwise
            # position bias mitigation strategy, options: swap_position, randomize_order, NULL. default is NULL
            position_bias_mitigation: NULL
            # completion parser config, default is lmsys_parser
            fn_completion_parser: lmsys_parser
    # prompt templates for auto reviewer(GPT-4)
    prompt_file: registry/data/prompt_template/prompt_templates.jsonl
    # target answer files list to be reviewed,
    # could be replaced by your own path: ['/path/to/answers_model_1.jsonl', '/path/to/answers_model_2.jsonl', ...]
    # Default is NULL, which means all answers in answers_gen will be reviewed
    target_answers: NULL
    # output file name of auto reviewer
    review_file: registry/data/arena/reviews/review_gpt4_2.jsonl

# rating results
rating_gen:
    enable: false
    metrics: ['elo']
    # elo rating report file name
    report_file: registry/data/arena/reports/elo_rating_origin.csv

```

---

## 执行评测

### 方式 1：命令行直接运行
```bash
```bash
python evalscope/run_arena.py \
  --c /path/to/cfg_arena.yaml  # 指定自定义配置文件路径
```

---

## 参数说明

| 参数              | 说明                                                                 |
|-------------------|--------------------------------------------------------------------|
| `--model`         | 模型标识符（HuggingFace格式或本地路径）                              |
| `--datasets`      | 评测数据集列表，支持多数据集空格分隔（如 `gsm8k arc`）                |
| `--limit`         | 每个数据集的最大评测样本数（默认全量评测）                            |
| `--c`             | 自定义配置文件路径（YAML格式）                                       |

---

## 注意事项
1. **显存管理**：大型模型建议使用 `gradient_checkpointing` 和 `zero_stage` 优化显存
2. **结果验证**：评测结果默认保存在 `./results/` 目录
3. **分布式支持**：可通过 `deepspeed` 配置多卡评测

更多文档参考：[EvalScope GitHub 仓库](https://github.com/modelscope/evalscope)