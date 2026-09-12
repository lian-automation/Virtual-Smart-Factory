# REPRODUCE.md — 数字复现指南

> 本仓库 README / 简历中的每一个数字，都由下表中的命令生成。
> 所有指标均为**仿真验证值**（如实标注，不冒充现场数据）。
> 环境：Windows 10/11 · Python 3.12 · `pip install -r requirements.txt`（numpy/flask/pymodbus）

## 一、质量门禁类（可精确复现，结果应完全一致）

| README 口径 | 生成命令 | 输出位置 / 预期 |
|---|---|---|
| 全厂自检 **17/17 通过** | `python selftest.py` | 控制台逐条 + `reports/selftest_report_*.txt`（A1~A9 模块级含有限料仓 + B1 联跑 + B2/B3/B4 冒烟 + C1~C4 算法/MES/EMS/订单生命周期） |
| **21 项 pytest**（17 selftest 转接 + 4 SCADA 网络冒烟） | `pip install -r requirements-dev.txt` 后 `pytest` | `tests/test_plant_selftest.py` + `tests/test_scada_network.py`（真实起停本地端口） |
| **ruff 静态检查零告警** | `ruff check .` | E4/E7/E9+F 基线，0 findings |
| **核心层覆盖率 74~84%** | `pytest --cov=core --cov=lines`（其余目录同理追加） | pytest-cov 覆盖率报告 |

## 二、长程压测类（统计口径复现，随机项会波动）

| README 口径 | 生成命令 | 说明 |
|---|---|---|
| **30 个仿真日**长跑 = 2592 万拍，墙钟约 9 分钟 | `python soak_run.py --days 30 --sample-min 60` | 可先用 `--sim-hours 0.5 --sample-min 10` 快速标定 |
| 流出 **79960 件** / 满托 **1572** / NG 率稳定 **5.6%** | 同上（该次运行实测值） | 故障注入为泊松随机，每次运行的件数等会在量级内波动 |
| 内存净增 **+25.7MB（斜率 ≈+0.9MB/仿真日）** | 同上，对比起止 RSS | 该斜率曾定位并修复一处内存线性增长缺陷，是 soak 的核心用途 |
| **2756 次故障注入账目全配对** / 托盘守恒零误差 | 同上（soak 末尾自动对账） | **守恒与配对应恒成立；若失败即真 bug**，欢迎以此验证 |

## 三、算法 A/B 类（数据源种子固定，可复现）

| README 口径 | 生成命令 | 说明 |
|---|---|---|
| 视觉算法 A/B：逻辑回归 **准确率 98.95% / 查全 84.8%**，对比规则法 **97.45% / 64.0%** | `python selftest.py`（C 模块自动执行三方对照） | 训练 1500 件 / 独立测试 2000 件，缺陷样本生成种子固定 |
| 良率 **96.4%**、OEE≈**90.0%**（A×P×Q 近似口径）、MES 用例报工 OK54/NG2 | `python selftest.py`（C 模块订单生命周期用例） | 48 件直灌 + 装配并行产出 |
| 历史报告示例 | `reports/selftest_report_20260825_145533.txt` | 运行时生成、不入库；重跑得到同格式新报告 |

## 四、在线演示类（交互验证）

| 功能 | 命令 | 入口 |
|---|---|---|
| 实时仿真 + 监控大屏 | `python main.py --web --speed 10` | 浏览器 <http://127.0.0.1:5080>（WebSocket :5081 实时推送） |
| Modbus TCP 从站（可连 MCGS 等组态软件） | 同上 | :1502，全设备 io_table 映射为保持寄存器；点表经 `/api/modbus/map` 导出；组态对接教程见 `docs/TUTORIAL_MCGS_OPENPLC.md` |
| MES 离线台账 + 回放对账 | `python mes/jsonl_replay.py` | 输出报工报表 + "回放总数 vs 各段行数合计"对账 |
| mes.db 保留策略 | `python tools/db_prune.py --max-mb 200`（参数可调） | 按最旧 run 分批清理 + VACUUM，至少保最新批次 |

## 五、口径说明（诚实边界）

- 本仓库**全软件仿真、零硬件依赖**；未经真机/产线验证，不冒充现场数据。
- 节拍/良率等设计值（如装配 32s、码垛 48 箱/托）来自 `config` 配置与 README 参数节；实测值来自上述命令的运行报告。
- 压测类数字随随机注入波动，但**账目自洽性（托盘守恒、故障配对）每次都必须成立**——这才是可复现的部分。
