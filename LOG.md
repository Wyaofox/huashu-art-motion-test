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

## 19:00 GIF 体积战（第二次全返工）

ffmpeg 直转 v1 结果全线超标：

| 文件 | v1 体积 | 问题 |
|---|---|---|
| 水墨写意 | 9.7MB | 贴线（限 10MB） |
| 莫奈 | **20.3MB** | 点彩纹理=高频细节，GIF 最坏情况 |
| Vox剪报 | **18.5MB** | 镜头平移+纸纹背景 → 全画面逐帧变化 |

调参三件套：`scale`（1080→960/900/848）+ `fps`（30→24/15/10）+ `max_colors`（256→160/128），调色板用 `palettegen=stats_mode=diff`（按运动区域优化）。v2 结果：**5.5 / 6.9 / 8.9MB**，全部达标，抽帧检查画质可接受。

## 结果

3/3 段产出成功，0 段失败。唯一与任务原设想的偏差：**测试图无法驱动 35 艺术风格**（引擎纯代码绘制，无图像风格迁移路径），已在 README 里如实说明。

## 遗留事项

- Gitee 仓库：凭据管理器里的旧 token 已失效（API 返回 401 Access token does not exist），需要新token 后创建。
- 上游可反馈点：winget/README 未提示 Windows 下 ffmpeg PATH 问题；官方 CDN 在国内网络不稳，可考虑文档里直接给 npmmirror 镜像方案。
