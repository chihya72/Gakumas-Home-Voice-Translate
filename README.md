# Gakumas Home Voice Translate

Windows 本地主页语音处理仓库：收录 ACB、复用或转换 WAV、通过 OpenMOSS 转写日语，再复用既有翻译引擎生成简体中文字幕。

## 使用

在 PowerShell 7 中运行：

```powershell
& (Join-Path 'D:\GIT\Gakumas-Home-Voice-Translate' 'run.ps1')
```

默认依次执行 collect、convert、asr、prepare、translate、export、status。再次运行仅收录尚未存在的 voiceAssetId，并补齐缺失 WAV、日语和中文。已有 ID 的源 ACB 更新不自动覆盖；需要重处理时由用户明确指定并维护相应数据。

也可以只运行一个阶段，例如：

```powershell
& (Join-Path 'D:\GIT\Gakumas-Home-Voice-Translate' 'run.ps1') -Stage status
& (Join-Path 'D:\GIT\Gakumas-Home-Voice-Translate' 'run.ps1') -Stage asr -Only @('sud_vo_system_amao_home_cmmn-03')
```

## 数据和依赖

- `tools\vgmstream-win64`：完整本地转换工具及许可证，纳入 Git。
- `audio\acb`：原始主页语音副本，纳入 Git；源文件不移动或删除。
- `audio\wav`：识别输入，保存在本地，Git 忽略。
- `data\name_dictionary.json`：完整导入的日中词典。ASR 仅使用全部日语键作官方热词提示，不按角色筛选、不注入角色卡或人物背景，不把中文译法混入转写，也不做训练或机械替换识别稿。每条原始识别记录保存实际使用的热词。
- `data\asr_hotwords.json`、`asr_prompt.txt`：实际使用的热词与提示。
- `data\asr_results.jsonl`：全新识别的原始输出、时间段和异常重识别记录，按尝试追加保存；早期调试记录也保留，当前采用的文本以 `voices.json` 为准。首次全量处理的最终 1549 条识别全部使用完整的 302 个字典热词。
- `data\voices.json`：按 voiceAssetId 关联音频、说话人、日语、中文和模型来源。可人工校对 ja、zh；已有非空值不会在普通增量运行中覆盖。
- `data\previous_user_subtitles.json`：单独保留旧任务的四条用户已有文本；新项目仍对全部音频重新识别。游戏中的现有字幕不修改。
- `translation\input`、`output`：现有引擎兼容的 CSV，保留完整语音 ID。
- `exports\home_voice_bilingual.json`、`.csv`：日中对照数据，CSV 可供校对。
- `exports\home_voice_subtitles.json`：当前 HV-10 使用的数组格式，text 为简体中文，保留原有 UI 配置。

外部依赖路径在 `config.json` 中配置。OpenMOSS 复用已有独立 Python、CUDA/BF16 模型和缓存，不复制模型。翻译直接加载 GakumasPreTranslation 的 `.env`、API 实现、模型、角色卡、术语表和只读翻译记忆，不复制密钥，不修改该仓库。按主表角色码读取说话人，主页语音使用独立 home_voice 分类，不注入连续剧情上下文。

ASR 默认每批 8 条，显存不足时自动拆分。翻译每批最多 250 条，同一批只包含一个角色；默认并行处理 3 个独立批次，可通过 `translation_concurrency` 调整。任一请求失败后停止分派新批次，让已经发出的请求保存成功结果。后续生成的 CSV 使用新的批次编号，保留此前输入与输出。

导出时将中文译文中机翻遗留的连续日语笑声 `ふふ` 规范为对应的 `呵呵`，将已发现的繁体字“傭”规范为“佣”，并在总表记录处理标记；日语识别原文和原始机翻 CSV 保留不变。角色名沿用既有配置，包括“手毬／小毬”的写法。

原始 ASR 与机翻均为自动稿，热词只能辅助模型，不保证姓名完全正确。解析失败、截断、整句重复或疑似输出中文的识别会使用更明确的转写提示单条重试，并以 1.1 的重复惩罚避免贪心解码陷入字符循环；重试仍使用全部字典热词，原始记录保留。使用 `-Stage asr -RedoAsr` 可明确重识别选中范围；原文发生变化时清空对应机翻，防止错误复用旧译文。翻译成功逐批保存，再次运行跳过已翻译行；失败请求不在外层原样重复。中文缺失时不会导出不完整的正式字幕文件。

ASR 在生成的完整片段结束时间达到 WAV 实测时长时停止，允许 0.12 秒的末尾时间戳误差；也会停止在剩余音频时长内明显不可能说完的未闭合尾段，只保留此前完整的转写片段，避免声音结束后继续凭热词生成内容。缺说话人标签但保留时间戳的输出也可解析。只清理与前段全文相同、至少 12 个日语字符却落在不超过 0.2 秒内的伪重复尾段，保留正常时长的重复发言。原始输出和完整分段均保留。

若第一次重试仍发生截断或整句循环，再进行一次有限的解码恢复：重复惩罚 1.2、禁止重复 8-token 片段，并要求用 `[laughs]` 标记笑声。正常识别不启用这项限制，所有尝试都保留完整字典热词和实际解码参数。

本项目仅生成字幕文件，不部署游戏文件、不启动或重启任何游戏或网页服务。未创建远程仓库。
