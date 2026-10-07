# 实测日志（2026-10-07）

按时间线记录，含每次失败的真实报错。

## 18:25 克隆与读文档

- `git clone --depth 1` 一次成功。
- 通读 `README.md` / `SKILL.md` / `render.py` / `clip.js` / `eras_gallery.js` / `clips/y2_vox.js` 后确认架构：
  - **skill 层**：SKILL.md + references/（35 张风格配方卡、9 张语法卡、12 篇方法文档）——给 coding agent 看的"怎么创作"知识。
  - **引擎层**：`scripts/engine/` 是可独立运行的完整工程。`render.py` 起本地 HTTP 服务 → 无头 Chromium 打开 `index.html`（整片）或 `clip.html`（spec 片段）→ 逐帧调 `window.renderFrame(t)` → canvas 像素以 PNG 管道喂 ffmpeg。
  - 两条渲染路径：`--solo <id>` 渲段落表里某个风格段；`--spec <json>` 渲参数化片段（时长严格=spec.duration）。
  - 依赖清单：uv、ffmpeg、Playwright Chromium。**无任何付费 API**（AI 生图仅用于角色帧，可完全绕开）。

## 18:35 装依赖（三连坑）

1. **uv**：`pip install uv` 进本地 venv，顺利。
2. **ffmpeg**：
   - gyan.dev 直链下载 → `curl: (56) schannel: server closed abruptly (missing close_notify)`，失败。
   - 换 BtbN GitHub release → 21s 后同样断连，失败。
   - `winget install Gyan.FFmpeg` → 成功。⚠️**坑**：winget 提示"添加了命令行别名 ffmpeg"，但 `WinGet/Links` 目录实际是空的；真正的 exe 在 `WinGet/Packages/Gyan.FFmpeg_Microsoft.Winget.Source_8wekyb3d8bbwe/ffmpeg-9.0.2-full_build/bin/`。
3. **Playwright Chromium**：
   - 官方 CDN 下载 → 下载中断（ChildProcess 报错栈），`ms-playwright/` 下没有 chromium 目录。
   - 换 npmmirror 镜像：`PLAYWRIGHT_DOWNLOAD_HOST=https://cdn.npmmirror.com/binaries/playwright` → 52s 下载 114.6MB 成功。

## 18:46 冒烟测试 ✅

README 自带命令：`--solo 09_postimp --stills 0.3`，一次出图。梵高段（少女+猫+星月窗+旋笔触）质量在线，链路 uv→Chromium→canvas→PNG 全通。

## 18:52 渲染正式段（踩坑：PATH）

- 第一跑 `--solo 17_ink --from 0 --to 1.172 --fps 60` → `FileNotFoundError: [WinError 2]`，来自 `_winapi.CreateProcess`——是 render.py 里 `subprocess.Popen(['ffmpeg', ...])` 找不到 ffmpeg。
- 原因：Git Bash 里给 PATH 追加的是 `C:/...` 风格路径，msys 不做转换，子进程解析不到。**改成 `/c/...` 风格后解决**。
- 渲染耗时：70 帧 37s ≈ **0.53 秒/帧**（1080p60）。

## 18:54 写 Vox spec（喂测试图）

- 关键机制（读 `render.py` 的 `ref()` + `clip.js` 得知）：spec 里图片路径相对 spec 文件，render.py 只把 spec 点名的文件加进白名单，经 `/__file__/` 路由给页面。
- `y2_vox` 的 cue 类型：`title`（黑底白字打字条）/ `image|clip`（贴图+胶带+纸白边）/ `point`（横线索引卡）/ `highlight`（`data.rect` 相对图片归一化，荧光笔扫过→红笔圈→镜头推近）。
- spec 写好后一次渲染通过：270 帧用 2m38s（0.59 秒/帧）。

## 19:00 GIF 体积控制

ffmpeg 直转第一版全线超标（莫奈 20.3MB、Vox 18.5MB——点彩纹理与镜头平移是 GIF 最坏情况）。调参三件套：`scale` + `fps` + `max_colors`，调色板用 `palettegen=stats_mode=diff`（按运动区域优化），全部压到达标且抽帧检查画质可接受。

## 20:05 迭代：风格变体路线（v2）

第一版两段 gallery 自绘段（17_ink / 28_monet）与测试图无关联，产出叙事不成立；改为"AI 风格变体＋引擎动画"路线：

1. ffmpeg 按 Vox 高亮框坐标精确裁切：`crop=307:205:276:369`（小舟）、`crop=538:461:799:307`（松亭）。
2. ImageGen 图生图 ×4：小舟 → 莫奈印象派 / 吉卜力；松亭 → 浮世绘 / 蒸汽波。实测构图保持良好，风格转换彻底。
3. 引擎 `y2_vox` 写两份 spec：`vox_boat.json`（原稿→莫奈→吉卜力）、`vox_pine.json`（原稿→浮世绘→蒸汽波），红线串联"同一画面的三种人生"。
4. 渲染 2×330 帧约 3.5 分钟/段；GIF 压缩后抽帧检查画质无退化。

这个"AI 出帧＋代码动画"的分工正是上游 skill 自己主张的路线（references/10-角色.md），实测走通。

## 20:20 v3：节奏调整与成片打包

v2 版文字停留太短（结尾卡只停 0.6s，标题不到 2s，不可读）。三份 spec 全部放慢——标题停留 ~3s、每张图 ~5s、结尾卡 ~2.5s，重渲 720/720/510 帧。

最终产物整理：

- `壁纸/`：5 张 PNG（原稿 1536×1024＋四张 1024×1024 风格变体），原画质不压缩。
- `视频/艺术动画实测_整体版.mp4`：65s，三段（整图剪报 17s → 小舟三变 24s → 松亭三变 24s）concat 无重编码拼接，1080p30 H.264。
- `视频/单段/`：三个单段 MP4。
- GIF 同步换 v3 慢节奏版（960px/12fps/192 色，22-25MB）。

## 遗留事项与上游反馈点

- Gitee 镜像仓库已建：https://gitee.com/Wyaofox/huashu-art-motion-test （旧 token 失效，更换新 token 后创建成功）。
- 建议上游文档补充：① Windows 下 ffmpeg 的 PATH 设置说明；② 官方 CDN 不稳时可直接给 npmmirror 镜像方案；③ winget 安装 ffmpeg 的实际 exe 位置提示。

## 结果

3/3 段产出成功，0 段失败。关键实测结论：**引擎无图像风格迁移能力**，"同一张图的不同风格版本"由生图模型在引擎外完成变体，引擎负责让它动起来——两个环节的分工即上游"人用 AI 生帧，代码负责合成"的路线，实测成立。
