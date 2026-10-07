<div align="center">

# huashu-art-motion-test

**对 [alchaincyf/huashu-art-motion](https://github.com/alchaincyf/huashu-art-motion)（花叔"艺术动画" Agent Skill）的一次全流程实测**

*用代码把画画出来再让它动 —— 3 段风格动画的诞生记录*

![License](https://img.shields.io/badge/license-MIT-green)
![Platform](https://img.shields.io/badge/platform-Windows-blue)
![Cost](https://img.shields.io/badge/付费_API-零-success)
![Status](https://img.shields.io/badge/实测-3%2F3_成功-brightgreen)

[简介](#简介) · [实测产出](#实测产出) · [快速复现](#快速复现) · [踩坑日志](docs/LOG.md) · [Gitee 镜像](https://gitee.com/Wyaofox/huashu-art-motion-test)

</div>

---

## 简介

[huashu-art-motion](https://github.com/alchaincyf/huashu-art-motion) 是花叔开源的 Agent Skill：让 coding agent 用代码"画"出 35 种艺术风格并让画面动起来，内置 9 种解说视频语法，画面全部由 Canvas 程序化绘制。

本仓库是对它的一次完整实测：**从克隆、装依赖、跑通渲染，到产出一组"同一幅 AI 水墨山水 → 四种画风变体"的动画成片**。所有真实报错、踩坑过程和最终命令都如实记录，可复现。

> **一句话结论**：它是"用代码把画画出来再让它动"的引擎，不是"把你的图变成动画"的风格迁移工具。风格变体交给生图模型，动起来交给引擎——与上游"人用 AI 生帧，代码负责合成"的分工一致，实测成立。

## 实测产出

3 段成片 GIF，均围绕同一张 AI 生成的水墨山水测试图：

| 文件 | 内容 | 时长 | 大小 | GIF 参数 |
|---|---|---|---|---|
| [`assets/gif/boat_variants.gif`](assets/gif/boat_variants.gif) | 同一叶小舟三种风格变奏：水墨 → 莫奈 → 吉卜力 | 24s | 25.2MB | 960px / 12fps / 192 色 |
| [`assets/gif/pine_variants.gif`](assets/gif/pine_variants.gif) | 同一座松亭三种风格变奏：水墨 → 浮世绘 → 蒸汽波 | 24s | 24.2MB | 960px / 12fps / 192 色 |
| [`assets/gif/shanshui_collage.gif`](assets/gif/shanshui_collage.gif) | 整幅山水贴上桌面 + 荧光笔高亮小舟/松亭两处 | 17s | 22.0MB | 848px / 10fps / 128 色 |

**素材链**：`assets/images/shanshui_original.png`（1536×1024 原稿）→ 按 spec 高亮框坐标裁出 `crop_boat.png` / `crop_pine.png` 两个局部 → ImageGen 图生图产出 4 张风格变体（`boat_monet` / `boat_ghibli` / `pine_ukiyo` / `pine_vapor`）→ 引擎 y2_vox 语法负责拼贴、红线、运镜、打字机字幕。spec 见 [`specs/`](specs/)。

> **v1 存档**：第一版两段引擎自绘场景（17_ink 水墨写意 / 28_monet 莫奈睡莲）移入 [`assets/v1/`](assets/v1/)，与最终路线对比用。

## 能力边界（实测结论）

- 35 种风格场景是**纯代码绘制**的（`scripts/engine/scenes/<id>.js`），引擎不接收外部图片做风格迁移——"喂一张山水画 → 输出梵高风格的山水动画"这条路径**不存在**。
- 外部图片能进入动画的两条路：`y2_vox`（剪报拼贴）和 `y3_whiteboard`（白板揭线）。
- 因此正确分工是：**风格变体交给生图模型（图生图），动起来交给引擎**——与上游"人用 AI 生帧，代码负责合成"的路线一致，实测走通。

## 环境依赖

| 组件 | 版本 | 获取方式 |
|---|---|---|
| [uv](https://docs.astral.sh/uv/) | 0.12+ | `pip install uv` |
| [ffmpeg](https://ffmpeg.org/) | 9.0 | winget（exe 实际在 `Packages/<包名>/.../bin/`） |
| Playwright Chromium | 1243 | `playwright install chromium` |
| Python | 3.13 | uv 自动管理 |

> **国内网络提示**：ffmpeg 与 Chromium 官方下载源容易断连。Chromium 走镜像：
> `PLAYWRIGHT_DOWNLOAD_HOST=https://cdn.npmmirror.com/binaries/playwright`

无需 Claude Code：skill 文档是给 agent 的方法包，`scripts/engine/` 可独立运行。

## 快速复现

```sh
# 0. 克隆上游项目
git clone --depth 1 https://github.com/alchaincyf/huashu-art-motion.git
cd huashu-art-motion

# 1. 冒烟测试：渲染梵高段一帧静帧
uv run --with playwright python scripts/engine/render.py --solo 09_postimp --stills 0.3 --out stills/

# 2. 单风格视频段（gallery 段每段 1.172s = 5 个八分音符 @128BPM）
uv run --with playwright python scripts/engine/render.py --film gallery --solo 17_ink \
  --from 0 --to 1.172 --fps 60 --out ink.mp4

# 3. JSON spec 驱动的参数化片段（图片路径相对 spec 文件）
uv run --with playwright python scripts/engine/render.py --spec specs/shanshui_collage.json --out vox.mp4

# 4. MP4 → GIF（scale + fps + max_colors 三件套，palettegen 按运动区域优化）
ffmpeg -i vox.mp4 -vf "fps=12,scale=960:-1:flags=lanczos,split[s0][s1];[s0]palettegen=stats_mode=diff:max_colors=192[p];[s1][p]paletteuse=dither=sierra2_4a" out.gif
```

渲染性能实测（1920×1080，CPU 软渲无 GPU）：**0.53–0.59 秒/帧**；24 秒片段约 3.5 分钟渲完。

## 目录结构

```
huashu-art-motion-test/
├── README.md                        # 本文件
├── CHANGELOG.md                     # 版本变更
├── LICENSE                          # MIT
├── docs/
│   └── LOG.md                       # 按时间线的完整实测日志（含真实报错）
├── specs/                           # y2_vox 参数化片段 spec（JSON）
│   ├── shanshui_collage.json        #   整图 + 双高亮
│   ├── boat_variants.json           #   小舟三变奏
│   └── pine_variants.json           #   松亭三变奏
└── assets/
    ├── gif/                         # 3 段最终成片 GIF
    ├── images/                      # 测试图原稿、局部裁切、4 张风格变体
    └── v1/                          # 第一版产出存档（引擎自绘段）
```

## 踩坑记录

完整时间线见 [docs/LOG.md](docs/LOG.md)。最疼的四个：

1. **下载源断连**：ffmpeg 直链（gyan.dev / GitHub release）与 Playwright 官方 CDN 全部中途掐断（`schannel: server closed abruptly`）→ winget 装 ffmpeg，Chromium 走 npmmirror 镜像。
2. **winget 假成功**：提示"已添加命令行别名"，但 `Links` 目录是空的，真正的 exe 在 `Packages/<包名>/.../bin/`。
3. **Git Bash PATH 隐形杀手**：PATH 追加 `C:/...` 风格路径，bash 本身不报错，但子进程解析不到——渲染器调 ffmpeg 直接 `WinError 2`，报错完全不指向根因。PATH 一律用 `/c/...` 风格。
4. **GIF 体积**：点彩纹理、镜头平移是 GIF 压缩最坏情况（直转 20MB+），`scale + fps + max_colors` 三件套 + `palettegen=stats_mode=diff` 解决。

## 致谢

- [alchaincyf/huashu-art-motion](https://github.com/alchaincyf/huashu-art-motion) —— 本仓库的实测对象，动画引擎与全部风格场景来自该项目（MIT）。
- 原项目始于对 [@cherry_mx_reds](https://x.com/cherry_mx_reds)《Art History Speedrun》的代码复刻。

## 许可证

[MIT](LICENSE) © 2026 Wyaofox

上游项目代码与文档同为 MIT；其角色形象（"少女+猫"）等素材另有使用限制，详见[上游 README](https://github.com/alchaincyf/huashu-art-motion#许可证)——`assets/v1/` 内的两段 GIF 含该形象，公开传播前请自行评估。
