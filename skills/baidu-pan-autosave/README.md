# 百度网盘自动转存 · WorkBuddy 技能

把「给出分享链接 + 百度 Cookie → 定时自动转存到本人网盘」这件事包装成可复用的 AI 技能。
底层复用开源项目 [`Jack261108/baidu-autosave`](https://github.com/Jack261108/baidu-autosave)，
运行在**你自己的 GitHub Fork** 上，由 GitHub Actions 定时调度，不需要本地机器常开。

## 这个技能解决什么问题

上游项目本身可用，但开箱即用会连续失败三次。本技能把修复与配置流程固化：

| 上游缺陷 | 后果 | 本技能的处理 |
|---|---|---|
| 工作流写死 Python 3.9，但 `requests==2.33.0` 要求 ≥3.10 | 装依赖直接失败 | 自动改为 3.12 |
| Python 3.12 起 `ensurepip` 不再自带 setuptools | Cython 扩展编译失败 | 自动补 `requirements.txt` |
| Fork 仓库的定时工作流默认 `disabled_fork` | 定时不触发 | 检测后调 API 启用 |
| `SAVE_DIR` 是父目录，容易理解错 | 产生嵌套重复目录，白占空间 | 默认 `/`，文档给出对照表 |

另外规避了一个只在 Git Bash 下出现的隐蔽问题：以 `/` 开头的参数会被 MSYS 改写成 Windows 路径，
导致百度报 `31062 文件名非法`。本技能的脚本全程用 `subprocess` + stdin 传值，不受影响。

## 前置条件

- 安装 [GitHub CLI](https://cli.github.com/) 并完成 `gh auth login`
- 令牌需包含 `repo` 与 `workflow` 两个 scope（后者用于提交工作流文件修改）
- Python 3.9+（脚本只用标准库，无第三方依赖）

## 安装

把本目录复制到 WorkBuddy 的用户级技能目录：

```bash
# Windows
xcopy /E /I skills\baidu-pan-autosave "%USERPROFILE%\.workbuddy\skills\baidu-pan-autosave"

# macOS / Linux
cp -r skills/baidu-pan-autosave ~/.workbuddy/skills/
```

重启对话后，技能列表中会出现 `baidu-pan-autosave`。

## 使用

```bash
# 查看现状：仓库、工作流状态、Secrets、最近运行的统计
python scripts/baidu_autosave.py status

# 首次搭建（Fork + 修复 + 启用）；已配置过的仓库重复执行是幂等的
python scripts/baidu_autosave.py provision --cron "17 10,11 * * *"

# 写入 Cookie 与分享链接
python scripts/baidu_autosave.py configure \
    --cookie-file C:/path/to/cookie.txt \
    --share-url "https://pan.baidu.com/s/1xxxxx?pwd=abcd" \
    --save-dir "/"

# 触发一次并等待结果
python scripts/baidu_autosave.py run
```

`all` 等同于 `provision` + `configure` + `run`。
仓库默认取「当前 gh 账号 / baidu-autosave」，可用 `--repo owner/name` 指定。

## Cookie 怎么获取

浏览器 Cookie 值带 `v20` 前缀，属 **App-Bound Encryption**。DPAPI 解密、CDP 调试端口、复制配置文件
三条自动化路径均已实测失败，只能手工复制：

1. 浏览器登录 `pan.baidu.com`
2. `F12` → **应用 / Application** → **Cookie** → `https://pan.baidu.com`
3. 筛选框输入 `BDUSS` 复制值；再输入 `STOKEN` 复制值
4. 拼成 `BDUSS=xxx; STOKEN=xxx`（分号 + 空格）

推荐存成本地文件后用 `--cookie-file` 传入，避免凭证进入对话记录。

## 使用前必须知道的三件事

1. **只补缺，不覆盖。** 同名文件若内容已变化（MD5 不同），会被记为「冲突」并跳过，不会替换。
   要更新只能先删除网盘里的旧文件，下一轮才补。
2. **`SAVE_DIR` 是父目录。** 填 `/` 才会落到 `/分享文件夹名`；填具体文件夹名会产生嵌套。
3. **Fork 后首次需人工点一次。** 打开 `https://github.com/<你>/baidu-autosave/actions`，
   点击横幅「I understand my workflows, go ahead and enable them」，之后才能用 API 启用。

更详细的排查记录见 [`references/pitfalls.md`](references/pitfalls.md)。

## 目录结构

```
skills/baidu-pan-autosave/
├── SKILL.md                      技能说明（触发条件与工作流）
├── README.md                     本文件
├── scripts/baidu_autosave.py     全部操作的确定性实现
└── references/pitfalls.md        踩坑记录与错误码速查
```

## 许可与致谢

本技能以 MIT 许可发布。上游项目 [`Jack261108/baidu-autosave`](https://github.com/Jack261108/baidu-autosave)
版权归原作者所有，本仓库是其 Fork，遵循同一 MIT 许可。
MIT 允许修改、再分发与商业使用，条件是保留原版权声明与许可条款。
