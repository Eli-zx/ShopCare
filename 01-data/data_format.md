# 数据格式与数据集说明

## 1. 分类数据格式

`train.txt`、`dev.txt`、`test.txt` 每行一条工单，文本与标签之间使用制表符分隔，多个标签使用英文逗号连接：

```text
快递到广州十天了还没动静，客服也不回复\tlogistics,service
电饭锅是坏的，我要退款，发票也没开\tquality,after_sale,invoice
```

约束如下：

- 文本与标签均不能为空，编码为 UTF-8。
- 标签必须来自 `class.txt`，顺序按 `class.txt` 的索引排列。
- 标签数量为 1 至 N 个；`invalid` 必须独占，不能与其他标签共现。
- 训练、开发、测试数据分别用于模型训练、阈值/校准和最终评估，测试集不参与调参。

## 2. 标签索引

| 索引 | 标签 | 含义 |
|---:|---|---|
| 0 | logistics | 物流配送 |
| 1 | quality | 商品质量 |
| 2 | after_sale | 退款退货 |
| 3 | invoice | 发票问题 |
| 4 | price_promo | 价格与优惠 |
| 5 | payment_account | 支付与账号 |
| 6 | consult | 售前与使用咨询 |
| 7 | service | 服务态度 |
| 8 | invalid | 无效文本 |

详细边界见 [`docs/标签规范.md`](../docs/标签规范.md)。

## 3. 当前数据规模

| 文件 | 数量 | 用途 |
|---|---:|---|
| `train.txt` | 100,000 | 模型训练 |
| `dev.txt` | 8,000 | 早停、概率校准和阈值标定 |
| `test.txt` | 8,000 | 最终评估 |

当前数据由 `tools/build_dataset_v2.py` 生成，仅用于复现链路。它使用模板家族分组切分、槽位隔离和短句覆盖，避免模板子句跨集合泄漏。`01-data/split_manifest.json` 保存了切分与泄漏检测结果。

旧脚本 `tools/generate_ticket_data.py` 只用于历史兼容，不建议用于指标实验，因为早期版本存在模板级泄漏风险。

## 4. 替换为真实数据

1. 将脱敏工单整理为 `文本\t标签1,标签2` 格式。
2. 使用 `python tools/check_dataset.py` 检查空行、未知标签、重复文本、长度和 `invalid` 独占约束。
3. 使用 `python 01-data/data_eda.py` 查看标签分布、共现关系和文本长度。
4. 按业务时间或模板家族切分 train/dev/test，避免同一工单、同一模板变体跨集合。
5. 重新训练模型，并在开发集上重新进行概率校准与阈值搜索。

公开评论数据通常是单标签，不能直接代表客服工单的复合诉求。将单标签升维为多标签时必须经过人工抽检，不能只依赖关键词规则。

## 5. 情感数据

`sentiment_class.txt` 固定使用 `positive`、`neutral`、`negative` 三类。目前线上情感分析由 `08-sentiment/sentiment_classifier.py` 的词典与规则引擎完成，不依赖情感训练集；后续如训练情感分类器，可使用 `文本\t情感标签` 的单标签格式。
