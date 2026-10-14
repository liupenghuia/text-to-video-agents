# 对白、镜头与口型同步架构

日期：2026-10-13。状态：Accepted（设计已确认，尚未实现）。关联 [范围](../product/scope.md)、[边界](./boundaries.md)、[门禁](./gates.md)、[ADR-004](../adr/ADR-004-dialogue-timeline.md)。本文定义后端存储、前端交互和调度共同使用的时间语义。

## 1. 已确认能力

- 首期为普通话角色对白剧情短剧，角色在画面说话时必须口型同步；普通后期配音不能替代此能力。
- 支持单角色和同镜头多角色轮流说话，不支持同时说话、抢话、合唱。不能静默降级为正反打，改变分镜须用户确认。
- 项目级 CharacterProfileVersion 固定形象与预设声音身份，跨镜头/跨 Episode 一致；每句情绪、语气、语速可变化并保留版本。上传样音、方言、多语言与旁白故事短片后续扩展。
- 用户确认后台词逐字保留；模型改写只能作为建议，确认新文本版本后再生成。漏词、改词、说话人错配属于硬失败。
- 对白与 Shot 不建立一对一关系。一句可跨镜头，一个镜头可有多句/多角色轮流对白。声音连续，画面切换不强制重做配音。
- 灵活、真实、自然的成片节奏是质量目标：镜头服务于表演和叙事，不为迁就台词边界机械切换。画外续声、听者反应、动作或环境镜头、自然停顿均须由确认的计划显式表达；自动判定未实测时由复核记录结论，不能假装已有客观阈值。

## 2. 正式流程与依赖

```text
剧本/台词确认
    ↓
绑定角色形象、预设声音与逐句表演参数
    ↓
生成对白音频 → 媒体探测 → 对齐/核对台词 → 归档
    ↓
Agent 按实际音频时长规划对白排布与镜头
    ↓
用户联合试听：对白 + 分镜 + 说话人区间 → 确认表演时间轴
    ↓
参考图/Keyframe → 视频及口型同步 → 候选审核 → 选用
    ↓
最终 Timeline 校验 → 渲染 → 成片核验与归档
```

Agent 可先用估算时长草拟分镜，正式确认和视频提交使用已归档音频的实际时长。音频准备也属于付费任务，需先预算预占。用户在联合预览调整声音/表演时，先生成新音频并重算时间，不能确认仍依赖旧时长的版本。

GenerationRun初始绑定剧本；同Run首次确认后续Speech/PerformancePlan时追加不可变阶段快照，是正常推进，不要求新建Run。替换已确认/执行过的输入才终止冲突意图并创建新Run，旧Job不重绑；草稿编辑不修改执行快照，见 [ADR-005](../adr/ADR-005-coding-baseline.md)。

原生音视频与独立 lip-sync 都可通过适配器实现，但必须消费确认的音频/时间契约。不能锁定文本后让视频模型自由生成另一段声音，再假定两者一致；真实 Provider 无法满足时保持阻断。

## 3. 后端数据关系与所有权

PostgreSQL 保存关系、时间、版本和审核；音频/视频/波形文件存私有对象存储，不将二进制媒体塞入 JSONB。

| 对象 | 所有者 | 核心字段及意义 |
|---|---|---|
| Character / CharacterProfileVersion | characters | projectId、characterId、version、形象参考 Asset、presetVoiceId、voiceProvider/model/config；确认版本不可变 |
| DialogueVersion | audio | episodeId、scriptVersionId、speakerProfileVersionId、text、order、language=zh-CN、performance 参数；不含唯一 shotId |
| SpeechAsset | audio | dialogueVersionId、assetId、actualDurationMs、sampleRate、alignmentVersion、文本核验结果、模型与配置快照 |
| SpeechAlignment | audio | speechAssetId、字符/词与音频时间对应、说话/静音区间、来源和置信度；无可信对齐不能宣布口型通过 |
| PerformancePlanVersion | timeline | episodeId、script/storyboardVersionId、revision、确认状态、确认人/时间；视频前的时间规划，不要求已有视频 Asset |
| AudioClip | timeline | planVersionId、speechAssetId、timelineStartMs、sourceStartMs、sourceEndMs、track/gain；映射音频源区间到 Episode 时间 |
| ShotPlacement | timeline | planVersionId、shotVersionId、timelineStartMs、durationMs；属于视频规划，不等于生成完成 |
| ShotCastBinding | storyboard | shotVersionId、characterProfileVersionId、画面人物标识/区域引用；区分谁出镜和谁说话 |
| SpeechVisualBinding | timeline | shotPlacementId、audioClipId、characterProfileVersionId、globalStart/EndMs、visibility=ON_SCREEN/OFF_SCREEN、lipSyncRequired |
| LipSyncTaskInput / LipSyncResult | jobs/对应执行模块 | Job 快照关联 plan/shot/speech/binding、目标区间、上下文区间、输入/输出 Asset 与映射；终态规则归 jobs |
| TimelineVersion / Render | timeline | 固定确认规划及选用视频/音频/字幕 Asset；最终可渲染版本，不覆盖预生成规划 |

关系用外键和唯一约束；核心时间列可查询并受范围校验；Provider 配置、表演参数、原始对齐 payload 可用有 Schema 的 JSONB。角色区域检测属于模型结果，人工修正后形成版本，不能靠出镜列表自动认定说话人。

SpeechAsset 是音频元数据，不复制 Asset 所有权：文件归 assets，语音时长/文本对齐归 audio。Shot 与 AudioClip 通过 SpeechVisualBinding 多对多，引用跨项目资源必须拒绝。按 planVersion、speakerProfile、speechAsset、shotPlacement 查询建立相应索引；具体迁移由实现阶段按既有风格生成。

## 4. 时间模型与实例

对外统一整数毫秒和半开区间 [start, end)。AudioClip 长度 = sourceEndMs - sourceStartMs；timelineEndMs = timelineStartMs + 长度。首期不支持音频时间拉伸；改语速必须重生成音频，不能只改时长字段。内部口型适配可按音频采样/视频帧换算，保留精度与映射，验收容差需实测配置。

一个音频片段全局覆盖 [0,6000)，两个镜头各 [0,3000)、[3000,6000)：

```json
{
  "planVersionId": "plan_v1",
  "audioClips": [
    {"id": "clip_a", "speechAssetId": "speech_a_v1", "timelineStartMs": 0, "sourceStartMs": 0, "sourceEndMs": 6000}
  ],
  "shotPlacements": [
    {"id": "p1", "shotVersionId": "shot_1_v1", "timelineStartMs": 0, "durationMs": 3000},
    {"id": "p2", "shotVersionId": "shot_2_v1", "timelineStartMs": 3000, "durationMs": 3000}
  ],
  "speechVisualBindings": [
    {"shotPlacementId": "p1", "audioClipId": "clip_a", "characterProfileVersionId": "a_v1", "globalStartMs": 0, "globalEndMs": 3000, "visibility": "ON_SCREEN", "lipSyncRequired": true},
    {"shotPlacementId": "p2", "audioClipId": "clip_a", "characterProfileVersionId": "a_v1", "globalStartMs": 3000, "globalEndMs": 6000, "visibility": "OFF_SCREEN", "lipSyncRequired": false}
  ]
}
```

p2 是听者 B 的反应镜头，A 的声音继续，B 不应跟随 A 动嘴；若 p2 拍 A，则对应 binding 改为 ON_SCREEN。OFF_SCREEN 必须来自明确分镜策略，不能为绕过 lip-sync 临时设置。

对某个 Shot 与 AudioClip：交集开始 s=max(shotStart,clipStart)，结束 e=min(shotEnd,clipEnd)。e<=s 无任务；否则源音频区间为 [clipSourceStart+s-clipStart, clipSourceStart+e-clipStart)，镜头局部区间为 [s-shotStart,e-shotStart)。再按可信说话/静音区间细分口型，停顿期间不强制动嘴。

多角色轮流说话时创建各角色 AudioClip 与 binding，按时间排列；确认对白片段不得重叠（首期明确安排的静音/音效可重叠）。对白之间自然停顿是计划数据，不由自动渲染拼接规则消除。

口型 Provider 如需上下文，可请求扩展源区间并记录 padding/trim 映射，超出源范围按适配器明确策略处理；结果只保留目标镜头时段，最终对白轨不拼接这些临时音频。禁止重复发声、遗漏切点音节或添加上下文时长。

## 5. 调度与渲染

- Speech请求按用户/操作/Idempotency-Key及payload去重；内容快照可复用已合格音频，但主动再次生成建立新意图/候选/预占，不能用同内容永久阻止新结果。保存真实音频/核验/对齐后推进SpeechReady。
- Storyboard/Planning Job 读取固定 SpeechAsset/Alignment，输出镜头、对白轨和视觉 binding，等待用户联合确认。
- 视频任务按镜头生成；LipSync 任务根据 ON_SCREEN binding 依赖镜头素材、音频和确认规划。单镜头多角色时必须使用多说话人适配或明确不冲突的局部处理策略，不能无验证地串行覆盖整帧导致前一个角色失去同步。
- 每个 lip-sync Job 保存精确输入区间和版本，重试不重新生成 Speech；不支持多角色处理时保持 BLOCKED，可建议正反打并由用户确认新分镜。
- Review 检查台词完整/说话人归属/声音身份/口型时序/非说话者错误动嘴/形象一致性，并检查跨镜头切点、连续对白、停顿、反应镜头与表演节奏是否符合确认计划；需要校准阈值，不宣称用简单 ASR 就能证明全部效果。自然度/衔接自动策略未实测时，保留人工复核证据而非编造评分。
- 最终渲染 AudioClip 的统一音轨。视频自带对白默认静音，环境音可独立保留，避免与主音轨叠加；替换主声音需重新确认 Speech 与表演计划。
- 最终字幕来自确认文本与真实 SpeechAlignment，通过 AudioClip 映射到全局时间，字幕跨 Shot 合法，不按镜头重复生成。

## 6. 前端展示与 API 能力

普通工作台提供联合确认视图，不要求首期高级多轨剪辑器：

- 角色列表展示形象/预设音色版本；台词列表支持原文、说话人、情绪语速、试听、音频状态、实际时长。
- 对白轨展示片段、停顿和字幕区间；镜头轨展示切点；说话人轨明确画内/画外和口型需求。点击片段联动台词、镜头和试听播放头。
- 同一句跨镜头时显示连续覆盖，不将片段拆成多个独立台词；单镜头多角色分别显示轮次。
- 联合确认提交 planVersionId + revision，展示预算、未解决依赖/审核及能力问题；确认前不发起相应视频生成。
- 更改声音/文本/表演参数展示影响预览并重生成 Speech，期间旧估算/时长明确标过期；冲突 409 保留草稿，不能把播放成功视作已通过口型审核。
- 桌面提供对齐轨道；窄屏改为镜头卡片与覆盖对白列表，保留试听、确认和错误入口；控件有键盘操作、标签和文本替代。

后端 OpenAPI 需提供角色档案、DialogueVersion/SpeechAsset 状态与试听授权、Speech 生成、规划查询/修改/确认、依赖影响预览、口型任务与审核状态。字段采用上述版本 ID 和时间单位；付费生成 202 + Job ID、Idempotency-Key、权限/预算门禁；确认原子检查 revision。具体 URL 在模块 API 实现时定义，不让前端凭 URL/时长自行构造权威关系。

## 7. 失效与验收

批量语音共享源文件时保存SpeechBatchVersion与成员源区间；重生成整段后整批对齐并重验消费者，不按后句文本未变认定偏移不变。DialogueOccurrence记录本次讲述/显式回放，跨Shot只是画面切换，不自动增加台词出现次数。源音轨完整性按计划实例与必需发声区间核对，详见 [ADR-005](../adr/ADR-005-coding-baseline.md)。

文本/声音/情绪语速变化 → 新 Dialogue/Speech → 新对齐 → 受影响规划/口型/字幕/Render 待更新；镜头切点或角色画内状态变化仅失效对应 binding/口型/渲染，不重做未改变 Speech。角色形象变化使相关视频失效，角色声音变化影响关联音频；跨 Episode 由显式选用档案版本控制，不全项目静默覆盖历史。

至少验证：一台词跨两 Shot、一 Shot 两角色轮流、反应镜头声音续播、画内跨切点同期、自然停顿不动嘴、切镜头不机械断句且保持叙事节奏、错人动嘴/改词硬失败、音频长度改变依赖失效、重复 Job/晚到结果、字幕跨镜头、无双重配音、用户确认正反打降级、取消恢复及资源越权。Mock 验证映射/调度，真实素材验证音画效果与自然度复核，两者不互相替代。
