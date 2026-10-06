# 配套脚本说明

这六个脚本覆盖"划分数据 → 训练 → 评估 → 误差分析 → 用未标注数据"的完整流程。各章节会告诉你什么时候用哪个。

## 使用前提

脚本会**自动向上查找作业根目录**（以 `data/`、`datasets/`、`codebase/` 这三个目录为标志），所以下面两种用法都能直接工作：

```bash
python code/train.py --name smoke --epochs 1     # 脚本留在 code/ 子目录
python train.py      --name smoke --epochs 1     # 或复制到作业根目录后运行
```

| 情况 | 做法 |
|---|---|
| 目录结构与教程一致（根目录下有 `data/`、`datasets/`） | 直接用上面任一种写法即可 |
| 数据或 codebase 放在别处 | 用 `--data`、`--project`、`--weights` 显式指定路径 |
| 想确认脚本认到了哪个根目录 | 脚本对 `--help` 无提示，但运行时的报错信息会打印它解析出的路径；也可先跑 `prepare_data.py --dry-run` 看它是否找到了图片 |

之所以做成自动查找，是因为有同学把脚本留在 `code/` 里、也有人习惯复制到根目录，两种都很常见；如果写死成 `Path(__file__).parent`，前一种用法会直接报"找不到图片"。

## 脚本清单

| 脚本 | 作用 | 关键参数 |
|---|---|---|
| `prepare_data.py` | 按框数分层划分数据，生成 `datasets/rm/` 与 `data.yaml`；图片用软链接，不复制文件 | `--dry-run` 只统计不写盘 |
| `train.py` | 训练；内置线程数限制与续训护栏；结束后自动打印可抄进作业表格的一行 | `--name` `--epochs` `--imgsz` `--batch` `--workers` `--device` `--fraction` `--resume` |
| `eval.py` | 在 train/val/test 上评估，输出 mAP50 / mAP75 / mAP50-95，并把指标落盘为 `metrics.json` 与总表 `summary.csv` | `--weights` `--split` `--imgsz` `--batch` |
| `analyze_errors.py` | 逐张误差分析：输出每张图的 TP/FN/FP 清单，并为漏检样本生成红蓝对比图 | `--weights` `--split` `--imgsz` `--conf` `--iou` |
| `autolabel.py` | 用已有模型给未标注数据生成预标注，并挑出最可疑样本供人工检查 | `--conf` `--imgsz` `--batch` `--sample` `--offset` `--limit` |
| `merge_addon.py` | 把（预）标注好的新数据并入训练集，**构建独立数据集**，原数据集不动、可随时回滚 | `--min-conf` `--min-boxes` `--drop-minimap` `--dry-run` |

## 典型顺序

```bash
python prepare_data.py                     # 1. 分数据
python train.py --name baseline --epochs 200 --imgsz 640 --batch 8 --workers 2   # 2. 训练
python eval.py --weights runs/baseline/weights/best.pt --split test --imgsz 640  # 3. 评估
python analyze_errors.py --weights runs/baseline/weights/best.pt --split test    # 4. 误差分析
python autolabel.py --conf 0.25            # 5. 未标注数据（可选）
python merge_addon.py --min-conf 0.5       # 6. 合并（可选）
```

## 两个设计说明

1. **续训护栏**：`train.py` 在续训时会读取断点里的 `imgsz`，若与命令行不一致直接报错退出。原因是输入尺寸属于"任务定义"而不是"性能参数"，续训时被静默改掉会让曲线断裂且难以察觉。
2. **合并到独立数据集**：`merge_addon.py` 不修改原数据集，而是生成 `datasets/rm_plus/`，因此实验结果不满意时删掉该目录即可回滚。
