# Changelog

本仓库所有重要变更都记录在这个文件里。
格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，版本号遵循 [语义化版本](https://semver.org/lang/zh-CN/)。

## [1.0.0] - 2026-10-07

### Added

- 首次全流程实测：环境搭建（uv / ffmpeg / Playwright Chromium）、3 段风格动画产出、渲染性能数据、踩坑日志。
- `assets/gif/`：3 段最终成片 GIF（小舟变奏、松亭变奏、Vox 剪报）。
- `assets/images/`：测试图原稿、2 张局部裁切、4 张 ImageGen 风格变体（莫奈 / 吉卜力 / 浮世绘 / 蒸汽波）。
- `specs/`：3 份 y2_vox 参数化片段 spec。
- `docs/LOG.md`：按时间线的完整实测日志。

### Changed

- v2：产出路线从"引擎自绘 gallery 段"调整为"AI 风格变体（ImageGen 图生图 ×4）+ 引擎 y2_vox 拼贴动画"。
- v3：spec 文字停留时长翻倍重渲（标题 ~3s / 图 ~5s / 结尾卡 ~2.5s），GIF 换 960px/12fps/192 色参数。

### Removed

- v1 产出的两段引擎自绘 GIF（17_ink / 28_monet）移入 `assets/v1/` 存档，不再作为正式产出。
