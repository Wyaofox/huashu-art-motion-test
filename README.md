# huashu-art-motion 实测记录

> 对 [alchaincyf/huashu-art-motion](https://github.com/alchaincyf/huashu-art-motion)（花叔"艺术动画"agent skill，2026-10 发布）的一次全流程实测：从 clone 到产出 3 段可用动画，全程踩坑如实记录。

## 一句话结论

**它是"用代码把画画出来再让它动"的引擎，不是"把你的图变成动画"的风格迁移工具**；画面全部由 Canvas 程序化绘制，零 AI 模型、零付费 API，普通电脑就能跑。

## 实测产出（3 段，v2 返工版）

| 文件 | 内容 | 画面来源 | 时长 | 大小 | 出片参数 |
|---|---|---|---|---|---|
| `assets/小舟变奏_01.gif` | 同一叶小舟 ×3 风格人生：水墨 → 莫奈 → 吉卜力 | 测试图局部（Vox 高亮框裁出）+ ImageGen 图生图变体 | 11.00s | 8.4MB | spec 1920×1080@30fps → 800px/10fps/112 色 GIF |
| `assets/松亭变奏_01.gif` | 同一座松亭 ×3 风格人生：水墨 → 浮世绘 → 蒸汽波 | 同上（松亭框） | 11.00s | 7.7MB | 同上 |
| `assets/Vox剪报_01.gif` | 整幅测试图入场 + 荧光笔扫过小舟/松亭两处 | 测试图整图 | 9.00s | 8.9MB | spec 1920×1080@30fps → 848px/10fps/128 色 GIF |

素材链全部围绕同一张测试图 `assets/test_shanshui.png`（1536×1024，AI 生成水墨山水）：`crop_boat.png` / `crop_pine.png` 为 Vox 高亮框坐标的精确裁切，`boat_monet.png` / `boat_ghibli.png` / `pine_ukiyo.png` / `pine_vapor.png` 为 ImageGen 图生图的 4 张风格变体；spec 见 `specs/`。

> **v1 存档**：第一版产出 `水墨写意_01.gif`（17_ink）和 `莫奈_01.gif`（28_monet）是引擎自绘的"少女+猫"场景，与测试图脱节，经反馈后按"AI 变体＋引擎动画"路线返工，两个文件保留在 `assets/v1_archive/` 供对比。

## 关于"用一张测试图跑 3 种风格"的如实说明

- 35 种艺术风格场景是**纯代码绘制**的（`scripts/engine/scenes/<id>.js`），引擎不接收外部图片做风格迁移——"喂一张山水画 → 输出梵高风格的山水动画"这条路径**不存在**。
- 外部图片能进入的两条路：`y2_vox`（剪报拼贴，图片贴上桌面+红线+荧光笔）和 `y3_whiteboard`（白板揭线）。实测走了 `y2_vox`，图片作为素材被贴进画面。
- 所以 3 段里只有 Vox 段真正"用"了测试图；水墨、莫奈两段是引擎自绘场景（选这两段是因为主题与山水最搭）。

## 环境（全部免费，无付费墙）

| 组件 | 版本 | 获取方式 |
|---|---|---|
| uv | 0.12.23 | pip 装进本地 venv |
| Playwright Chromium | 1243（153.0.8010.12 headless） | npmmirror 镜像下载 114.6MB |
| ffmpeg | 9.0.2 full build（gyan.dev） | winget |
| Python | 3.13（uv 自动管理） | — |

无需 Claude Code：skill 文档（SKILL.md + references/）是给 agent 的方法包，`scripts/engine/` 可独立运行。

## 渲染性能实测

1920×1080 逐帧渲染：60fps 段 70 帧 ≈ 37s（**0.53 秒/帧**）；30fps 段 270 帧 ≈ 158s（0.59 秒/帧）。CPU 软渲，无 GPU 依赖。

## 完整踩坑日志

见 [LOG.md](LOG.md)。

## 复现命令

```sh
# clone
git clone --depth 1 https://github.com/alchaincyf/huashu-art-motion.git

# 单风格静帧（检查用）
uv run --with playwright python scripts/engine/render.py --film gallery --solo 17_ink --stills 0.5 --out stills/

# 单风格视频段（gallery 段每段 1.172s = 5 个八分音符 @128BPM）
uv run --with playwright python scripts/engine/render.py --film gallery --solo 17_ink --from 0 --to 1.172 --fps 60 --out ink.mp4

# JSON spec 驱动的参数化片段（图片路径相对 spec 文件）
uv run --with playwright python scripts/engine/render.py --spec vox_shanshui.json --out vox.mp4

# MP4 → GIF（体积控制三件套：scale + fps + max_colors）
ffmpeg -i ink.mp4 -vf "setpts=2*PTS,fps=24,scale=960:-1:flags=lanczos,split[s0][s1];[s0]palettegen=stats_mode=diff:max_colors=160[p];[s1][p]paletteuse=dither=sierra2_4a" 水墨写意_01.gif
```

## 许可证提醒 ⚠️

gallery 场景画面里有花叔的"少女+猫"角色形象。上游 README 许可证节写明：**该形象"只用于本 skill 的示范，不随 MIT 授权用于其他用途"**。本文仓库的 GIF 属实测记录；若要把含该形象的片段用于公众号等公开传播，需自行评估。`Vox剪报_01.gif` 用的是自产测试图，无此问题。
