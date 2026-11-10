# AGENTS.md

本文件面向在本仓库中工作的 AI 助手与贡献者，说明项目结构、运行方式与必须遵守的约束。

## 项目概述

**Trae AutoCheckin** 是一套 Trae 每日签到与积分监控工具，包含 4 个可独立运行的 Python 脚本：

| 脚本 | 作用 |
| :--- | :--- |
| `trae_checkin.py` | 主脚本：多账号签到、Token 续期、9074 换设备号重签、推送 |
| `trae_credit_monitor.py` | 只读积分监控（绝不签到），支持积分包额度解析 |
| `trae_sms_login.py` | 通过官网真实 OAuth 网页流程换取 refreshToken（本地回调端口 17388） |
| `trae_get_token.py` | 从本机 Trae 客户端读取并解密凭据，支持 `--accounts` / `--export-keys` |

支持青龙面板、本地 crontab、GitHub Actions 三种部署方式。

## 技术栈与硬性约束

- **Python 3.8+，零第三方依赖**：只允许使用标准库。不要引入 `requests`、`cryptography` 等包；需要的能力（AES、ECDSA P-256、PKCE、HTTP）已在脚本内手写实现。
- **单文件脚本**：每个脚本独立可运行，不建立包结构，不拆分公共模块。需要复用逻辑时在各脚本内部实现。
- 代码风格跟随现有文件：4 空格缩进、顶层函数式写法、`snake_case` 命名、中文日志与注释。
- 保持与 `trae_checkin.py` 现有实现一致的接口风格。

## 目录结构

```text
.
├── trae_checkin.py            # 签到主流程
├── trae_credit_monitor.py     # 积分只读查询
├── trae_sms_login.py          # 网页 OAuth 登录
├── trae_get_token.py          # 本地凭据提取 / 解密
├── tests/test_trae_regressions.py  # 离线回归测试
├── docs/                      # 多语言 README
├── .github/workflows/daily.yml     # GitHub Actions 定时任务
└── README.md                  # 主文档（环境变量、FAQ、原理）
```

## 常用命令

```bash
# 运行签到（需先配置 TRAE_ACCOUNTS / TRAE_REFRESH_TOKEN 等环境变量）
python3 trae_checkin.py

# 只读查询积分
python3 trae_credit_monitor.py

# 提取本机凭据 / 导出多账号数组 / 导出设备密钥
python3 trae_get_token.py
python3 trae_get_token.py --accounts
python3 trae_get_token.py --export-keys

# 网页 OAuth 登录（--solo 走 SOLO 产品线，--selftest 离线自检）
python3 trae_sms_login.py

# 跑全部离线回归测试（无需网络与真实 token）
python3 -m unittest discover -s tests -v
```

## 核心流程与关键概念

`trae_checkin.py` 主链路（详见 README「工作原理」）：

1. `load_accounts()` 读取 `TRAE_ACCOUNTS`（优先）或 `TRAE_REFRESH_TOKEN[_N]`。
2. `ensure_token()` 复用缓存中的有效 `accessToken`，过期或 `code=1001` 时调用 `refresh()` 续期。
3. `refresh()` 走带设备证明的 `ExchangeToken`（`device_proof()` 用 ECDSA P-256 签名），成功后 `persist_tokens()` **链式回写**新的 refreshToken。
4. `status_of()` 查询今日是否已签，未签则 `claim_with_retry()` 领取。
5. 命中 `9074` 时由 `rotate_device_id()` 换真实设备号重签；仍失败才进入冷却（`set_cooldown()`）。
6. `push()` 通过企业微信 / PushPlus 推送结果。

必须理解的业务约定：

- **`9004` 与 `9095` 都表示「今日已签到」**，不是失败。
- **`9074` 基本等于「设备号不对」**：只能用客户端注册过的真实设备号（`real_device_ids()`），禁止生成假号硬撞。
- **签到积分不叠加**：`reward_credits()` 优先取正的 `credits`，否则取正的 `extra_credits`，两者绝不相加。
- **refreshToken 是轮换链**：每次续期产生新值、旧值立即失效。任何改动都必须保证新 token 回写到缓存与账号文件。
- 状态文件（`.trae_checkin_state.json`）用于当日跳过已签账号与冷却控制。

## 不得触碰的红线

- 绝不提交任何真实凭据：`.trae_token_cache.json`、`.trae_checkin_state.json`、`.trae_device_ids.json`、`trae_sms_accounts.json`、`storage.json`、`*.pem`、`.env*` 均已在 `.gitignore` 中，不要绕过。
- 不要修改 `assets/`、`LICENSE`、`NOTICE.md` 中的作者与版权信息。
- 不要在日志、测试或异常信息中打印完整 token / 私钥。
- 不要调整 README 中的使用边界与免责声明。

## 测试要求

- 测试为纯离线 `unittest`，通过 `importlib.util.spec_from_file_location` 直接加载脚本，不允许真实网络请求或真实凭据。
- 新增行为（尤其是业务码处理、积分计算、批次选择）必须补充 `tests/test_trae_regressions.py` 中的回归用例。
- 修改后至少运行一次：`python3 -m unittest discover -s tests -v`，确保全部通过。

## 文档同步

- 新增/修改环境变量时，同步更新 `README.md` 的「环境变量」表格与脚本头部 docstring。
- 版本号在 `README.md` 与 `CHANGELOG.md` 中保持一致。
