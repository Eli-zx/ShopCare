# ShopCare：电商客服工单智能处理平台

ShopCare 是一个面向电商客服场景的中文工单智能分流平台。系统将一条用户反馈转换为可执行的业务决策：**多标签诉求、情感极性、优先级与 SLA、责任部门、建议回复，以及是否需要人工复核**。

## 项目概述

面向电商客服工单的多标签文本分类与智能分流平台。基于 FastAPI 搭建统一服务，支持 TF-IDF + OneVsRest 随机森林、FastText、BERT + 手写 LoRA 三套可切换模型；通过难例动态加权采样、概率校准、双阈值拒识和可选外部推理服务处理低置信度样本；结合情感分析、优先级规则、话术审批闸门与人工复核队列，形成从模型推理到工单处理的完整链路。

### 核心实现

- 将单标签分类改造成 9 类多标签任务，使用独立 sigmoid 与 BCEWithLogitsLoss，支持“质量 + 退款 + 服务”等复合诉求。
- 实现手写 LoRA，仅向 BERT 的 query/value 注入低秩适配器，并冻结主干参数，降低训练和部署成本。
- 设计难例动态加权采样，依据标签数量、漏判率和置信度缺口调整样本权重，重点学习混合诉求与低置信度样本。
- 设计单标签阈值 + 全局平均置信度的双阈值拒识，形成“本地模型 → 可选外部推理 → 人工复核”的可解释升级链路。
- 统一三套模型的数据读取、评估和推理返回契约，并用概率校准改善跨模型置信度可解释性。
- 使用 FastAPI、JWT、Redis/MySQL 可选存储、限流、缓存、审计日志和原生前端，完成可运行的端到端业务闭环。

## 当前实验结果

以下结果来自仓库内的合成演示语料：训练集 100,000 条，开发集 8,000 条，测试集 8,000 条。数据使用模板家族分组切分和槽位隔离，测试集子句在训练集中的原样出现比例为 0%。指标用于验证工程链路和模型相对差异，不代表真实业务效果。

| 模型 | 测试 Micro-F1 | Macro-F1 | 整单准确率 | 自动分流率 | 结果文件 |
|---|---:|---:|---:|---:|---|
| TF-IDF + OneVsRest 随机森林 | 0.6533 | 0.6536 | 0.3645 | 87.94% | `02-rf/result/metrics_rf.json` |
| 多标签 FastText | 0.7597 | 0.7825 | 0.4694 | 83.53% | `03-fasttext/result/metrics_fasttext.json` |
| BERT + LoRA | 0.9175 | 0.9275 | 0.8365 | 89.88% | `04-bert/result/test_metrics.json` |

BERT 指标来自 2026-09-22 的云端 GPU 训练；本机 CPU 配置与该实验不同，不能直接复现同一数字。所有指标和阈值都应以对应 JSON 结果文件为准。

## 快速开始

```bash
pip install -r backend/requirements.txt
python tools/check_dataset.py
python tools/check_env.py
python tools/build_dataset_v2.py
python 02-rf/rf_train.py
python 03-fasttext/ft_train.py
python 04-bert/train_bert.py
python backend/main.py
```

浏览器访问 `http://127.0.0.1:8000/app`，接口文档访问 `/docs`，健康检查访问 `/health`。MySQL 和 Redis 不可用时，系统会分别降级到内存存储和进程内缓存/限流，并在健康检查中标明降级状态。

## 项目结构

```text
01-data/                 数据格式、标签定义、训练/开发/测试数据
02-rf/                   TF-IDF + OneVsRest 随机森林
03-fasttext/             多标签 FastText
04-bert/                 BERT + 手写 LoRA + 难例采样 + 拒识
08-sentiment/            词典与规则情感分析
09-priority/             优先级、SLA 与人工复核规则
10-reply-recommend/      回复模板与审批闸门
11-ticket-kg/            标签共现与风险组合分析
backend/                 FastAPI、认证、存储、模型路由与业务编排
frontend/web/            零构建原生 HTML/CSS/JavaScript 前端
tools/                   数据构建、校验、校准和全链路自检
docs/                    环境、标签、设计与改造说明
```

## 核心决策链路

1. 模型输出 9 个标签的独立概率，不使用 softmax。
2. 单标签阈值筛选激活标签，再用全局平均置信度判断是否自动分流。
3. 低置信度工单进入拒识流程；开启开关时调用兼容接口进行外部推理，否则进入人工复核队列。
4. 对通过分类的工单继续执行情感、优先级、SLA、责任部门和建议回复生成。
5. 结果保存为工单决策快照，支持复核回写、审计和看板统计。

## 9 类标签

`logistics` 物流配送、`quality` 商品质量、`after_sale` 退款退货、`invoice` 发票问题、`price_promo` 价格与优惠、`payment_account` 支付与账号、`consult` 售前与使用咨询、`service` 服务反馈、`invalid` 无效文本。

`invalid` 为独占标签，其余标签允许组合。标签中文名、责任部门、基础优先级和 SLA 的唯一事实来源是 `01-data/label_meta.json` 与 `01-data/class.txt`。

## 文档导航

| 文档 | 用途 |
|---|---|
| [`01-data/data_format.md`](01-data/data_format.md) | 数据格式、切分策略和真实数据替换 |
| [`docs/环境安装指南.md`](docs/环境安装指南.md) | 环境安装、模型准备、服务启动与自检 |
| [`docs/标签规范.md`](docs/标签规范.md) | 标签边界、共现约束和标注流程 |
| [`docs/改造说明.md`](docs/改造说明.md) | 从教学项目到当前平台的改造记录 |
| [`docs/客服工单用户反馈文本分类/ShopCare-设计文档.md`](docs/客服工单用户反馈文本分类/ShopCare-设计文档.md) | 当前架构与模块设计 |
| [`docs/客服工单用户反馈文本分类/客服工单用户反馈文本分类——落地级NLP项目详细方案.md`](docs/客服工单用户反馈文本分类/客服工单用户反馈文本分类——落地级NLP项目详细方案.md) | 已实现能力与后续规划的边界 |

## 已知限制

- 当前数据为合成演示语料，真实业务效果需要使用脱敏工单重新训练和标定。
- BERT 的预训练模型需要本地准备；项目不联网下载模型。
- 05/06/07 模型压缩目录尚未实现，不应作为当前项目成果描述。
- 前端暂未做权限分级，正式部署仍需补充单元测试、CI、审计归档和多实例 Redis 方案。
