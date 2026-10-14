# Agent 文生视频短剧系统完整执行流程

> 文档状态：历史架构流程提案，保留完整运行链路作为参考。当前完整产品的实施基线见 [架构与实施 TODO](../../todo.md)；文中的旧目录、未决技术和 MVP 范围需逐步对齐新决策，不代表已实现能力。
>
> 对白/口型部分已由 [音画时间轴规范](./dialogue-timeline.md) 与 [ADR-004](../adr/ADR-004-dialogue-timeline.md) 更新：首期普通话角色短剧，先实际语音，再按时长规划和联合确认分镜，最后视频/口型。不能沿用“视频全部完成后才首次 TTS”的旧执行顺序。
>
> 本文描述系统目标运行链路和工程分工，不代表所有技术选择已经最终确定。关于调研与记录方法，请参阅 [项目调研与决策记录指南](../项目调研与决策记录指南.md)；哪些内容已确定、哪些还需要验证，以当前 TODO 和后续已接受 ADR 为准。
>
> 文档分工：本文写“系统应该怎样运行”；调研指南写“哪些地方还需要验证和选择”；最终架构选择应记录在 ADR 中，并在决策确定后回写本文。

## 0. 先建立一个最重要的认知

整套系统不是：

```text
用户
 ↓
Agent
 ↓
视频模型
 ↓
MP4
```

真正应该是：

```text
用户
 ↓
API Server
 ↓
Workflow
 ↓
Agent / Tool
 ↓
数据库 + Queue
 ↓
Worker
 ↓
第三方 AI 服务
 ↓
Asset 存储
 ↓
下一个 Workflow 节点
 ↓
最终 Render
```

其中每个角色解决的问题完全不同。

```text
Agent
负责“想”：
生成什么内容、怎么拆镜头、Prompt 怎么写、质量是否合格。

Workflow
负责“流程”：
现在执行哪一步，下一步是谁，失败后去哪。

Queue
负责“排队”：
视频生成任务很多，不能一起把服务器打爆。

Worker
负责“干活”：
真正调用 LLM、图片 API、视频 API、FFmpeg。

PostgreSQL
负责“记住业务状态”：
项目、场景、Shot、任务状态、Asset 元数据。

Redis
负责“临时任务调度”：
Queue、Lock、缓存。

OSS / S3
负责“存大文件”：
图片、视频、音频、字幕、最终成片。

第三方 AI Provider
负责“AI 推理”：
LLM、生图、生视频、TTS、Vision。
```

一句话：

> **Agent 决策，Workflow 调度，Worker 执行，数据库记状态，OSS 存素材，模型负责生成。**

---

# 1. 用户发起一个短剧生成请求

假设用户在前端输入：

```text
生成一个 20 秒悬疑短剧。

剧情：
男人回到家，
走到卧室门口，
握住门把手，
缓慢打开门，
发现房间里站着一个陌生人。

风格：
写实电影感、夜晚、冷色调。

比例：
9:16
```

前端点击：

```text
生成短剧
```

此时前端调用：

```http
POST /api/projects
```

请求：

```json
{
  "idea": "男人回家后打开卧室门，发现陌生人",
  "duration": 20,
  "style": "cinematic realistic thriller",
  "aspectRatio": "9:16"
}
```

---

# 2. API Server 第一件事：不是调用 AI

很多 Demo 这里直接：

```typescript
await openai(...)
```

生产系统不应该这样。

第一件事情应该：

```text
创建 Project
```

PostgreSQL：

```sql
INSERT INTO projects
```

数据：

```json
{
  "id": "project_001",
  "title": null,
  "idea": "男人回家后打开卧室门，发现陌生人",
  "duration": 20,
  "style": "cinematic realistic thriller",
  "aspect_ratio": "9:16",
  "status": "CREATED"
}
```

为什么必须先存？

因为从现在开始：

```text
任何一步都可能失败
```

例如：

- LLM 超时
- 视频生成失败
- 服务重启
- 用户刷新页面
- Worker 崩溃
- Redis 重启

Project 是整个流程的“锚点”。

---

# 3. 创建第一个 Job

Project 创建成功后：

```text
Workflow 判断：

下一阶段 = SCRIPT_GENERATION
```

于是数据库插入：

```text
jobs
```

例如：

```json
{
  "id": "job_script_001",
  "project_id": "project_001",
  "type": "SCRIPT_GENERATION",
  "status": "PENDING",
  "attempt": 0
}
```

然后：

```text
BullMQ
```

推入：

```text
script-queue
```

Payload：

```json
{
  "jobId": "job_script_001",
  "projectId": "project_001"
}
```

注意一个非常重要的原则：

> Queue 尽量不要塞完整业务数据，只塞 ID。

不要：

```json
{
  "整个剧本": "...",
  "所有人物": "...",
  "所有图片URL": "..."
}
```

Worker 应该凭：

```text
projectId
```

自己去数据库取最新数据。

这样不会产生数据版本错乱。

---

# 4. Script Worker 开始工作

现在：

```text
script-worker
```

监听：

```text
script-queue
```

收到：

```json
{
  "jobId": "job_script_001",
  "projectId": "project_001"
}
```

第一件事情：

```sql
SELECT * FROM projects
WHERE id = 'project_001';
```

得到用户需求。

然后：

```text
job.status
PENDING
  ↓
RUNNING
```

写数据库：

```sql
UPDATE jobs
SET status='RUNNING'
```

---

# 5. Script Agent 此时才出现

Script Worker 调用：

```text
ScriptAgent
```

这里 Agent 并不直接控制数据库。

推荐：

```text
Worker
   ↓
Agent
   ↓
LLM
```

Agent 输入：

```json
{
  "idea": "男人回家打开卧室门发现陌生人",
  "duration": 20,
  "style": "cinematic realistic thriller",
  "aspectRatio": "9:16"
}
```

Agent 给 LLM 的 System Prompt 大概：

```text
你是影视短剧编剧。

你的任务不是输出小说，
而是输出适合视频生成系统处理的结构化剧本。

总时长控制在20秒左右。

必须包含：

characters
scenes
dialogues
actions
dramatic beats
```

---

# 6. 为什么这里必须 Structured Output

LLM 不应该返回：

```text
夜色深沉。

张默缓缓走到门前……
```

应该返回：

```json
{
  "title": "门后的人",
  "characters": [
    {
      "id": "char_001",
      "name": "张默",
      "gender": "male",
      "age": 32,
      "appearance": "黑色短发，灰色夹克，黑色长裤"
    },
    {
      "id": "char_002",
      "name": "神秘人",
      "gender": "male",
      "age": 40,
      "appearance": "黑色风衣，站在阴影中"
    }
  ],
  "scenes": [
    {
      "id": "scene_001",
      "location": "apartment corridor",
      "time": "night",
      "description": "张默走到卧室门前"
    },
    {
      "id": "scene_002",
      "location": "bedroom",
      "time": "night",
      "description": "门打开，陌生人站在房间里"
    }
  ]
}
```

这样后面的程序才能：

```typescript
script.characters
script.scenes
```

直接读取。

而不是重新解析自然语言。

---

# 7. Script Worker 收到结果后干什么

LLM 返回数据后：

```text
Script Worker
```

开始持久化。

数据库：

```text
characters
```

写：

```json
{
  "id": "char_001",
  "project_id": "project_001",
  "name": "张默",
  "appearance": "...",
  "clothing": "灰色夹克"
}
```

然后：

```text
scenes
```

写：

```json
{
  "id": "scene_001",
  "project_id": "project_001",
  "location": "apartment corridor",
  "time": "night"
}
```

Script JSON 本身也可以存：

```text
scripts
```

或者 JSONB。

然后：

```text
job_script_001
RUNNING
 ↓
SUCCEEDED
```

---

# 8. Script 完成以后谁决定下一步？

不是 Script Agent 决定。

应该：

```text
Workflow Engine
```

得到事件：

```text
SCRIPT_GENERATION_COMPLETED
```

Workflow 判断：

```text
下一个阶段：
STORYBOARD_GENERATION
```

于是：

```text
创建 storyboard job
```

↓

```text
推入 storyboard-queue
```

---

# 9. Storyboard Worker 取数据

Storyboard Worker 收到：

```json
{
  "projectId": "project_001"
}
```

然后数据库查询：

```text
project
script
characters
scenes
```

此时不是去 Redis 取剧本。

Redis 不是业务数据库。

应该：

```text
PostgreSQL = Source of Truth
```

---

# 10. Storyboard Agent 开始拆 Shot

Script：

```text
张默走到门前，
握住把手，
缓慢打开门，
看见陌生人。
```

Storyboard Agent 输出：

```json
[
  {
    "id": "shot_001",
    "scene_id": "scene_001",
    "sequence": 1,
    "duration": 4,
    "shot_type": "medium",
    "camera": "slow tracking",
    "action": "张默走向卧室门",
    "transition": "continuous"
  },
  {
    "id": "shot_002",
    "scene_id": "scene_001",
    "sequence": 2,
    "duration": 4,
    "shot_type": "close_up",
    "camera": "static",
    "action": "张默伸手握住门把手",
    "transition": "continuous"
  },
  {
    "id": "shot_003",
    "scene_id": "scene_001",
    "sequence": 3,
    "duration": 5,
    "shot_type": "medium_close_up",
    "camera": "slow push",
    "action": "张默缓慢打开房门",
    "transition": "continuous"
  },
  {
    "id": "shot_004",
    "scene_id": "scene_002",
    "sequence": 4,
    "duration": 5,
    "shot_type": "over_shoulder",
    "camera": "static",
    "action": "门后出现神秘人",
    "transition": "cut"
  }
]
```

现在第一次出现一个非常重要的数据：

```text
transition
```

它会直接影响以后：

```text
Shot2 是否使用 Shot1 最后一帧
```

---

# 11. Shot 写进数据库

数据库：

```text
shots
```

Shot1：

```json
{
  "id": "shot_001",
  "project_id": "project_001",
  "scene_id": "scene_001",
  "sequence": 1,
  "duration": 4,
  "transition": "continuous",
  "status": "PLANNED"
}
```

Shot2：

```json
{
  "id": "shot_002",
  "previous_shot_id": "shot_001",
  "transition": "continuous",
  "status": "PLANNED"
}
```

Shot4：

```json
{
  "id": "shot_004",
  "previous_shot_id": "shot_003",
  "transition": "cut"
}
```

注意：

```text
Shot4 = cut
```

意味着：

> 不应该使用 Shot3 最后一帧。

---

# 12. 下一阶段：角色资产生成

Workflow：

```text
STORYBOARD_COMPLETED
 ↓
CHARACTER_ASSET_GENERATION
```

创建：

```text
image jobs
```

例如：

```text
char_001_reference
char_002_reference
```

推入：

```text
image-queue
```

---

# 13. Image Worker 做什么

Image Worker：

```text
读取 character
```

例如：

```json
{
  "name": "张默",
  "age": 32,
  "appearance": "亚洲男性，黑色短发",
  "clothing": "灰色夹克，黑色长裤"
}
```

然后调用：

```text
Prompt Compiler
```

注意：

```text
Prompt Compiler
```

不是 Agent。

它最好是普通代码。

比如：

```typescript
function compileCharacterPrompt(character) {
    return `
    cinematic realistic portrait,
    East Asian male,
    age ${character.age},
    ${character.appearance},
    wearing ${character.clothing},
    neutral pose,
    consistent identity
    `;
}
```

---

# 14. 然后才调用 Image Provider

统一接口：

```typescript
imageProvider.generate(...)
```

底层可以：

```text
OpenAI Image
Midjourney
Flux
SDXL
其他服务
```

返回图片。

例如：

```text
/tmp/char_001.png
```

---

# 15. 为什么要立即上传 OSS

Worker 本地磁盘：

```text
/tmp/
```

属于临时环境。

服务器重启：

```text
文件没了
```

所以生成完成：

```text
立即上传 OSS/S3
```

变成：

```text
https://cdn.xxx.com/
projects/project_001/
characters/char_001/reference.png
```

然后数据库写：

```text
assets
```

：

```json
{
  "id": "asset_char_001",
  "project_id": "project_001",
  "type": "IMAGE",
  "purpose": "CHARACTER_REFERENCE",
  "url": "https://...",
  "provider": "xxx"
}
```

数据库存：

```text
元数据 + URL
```

OSS 存：

```text
真正文件
```

---

# 16. Shot1 开始准备

现在：

```text
角色参考图
场景数据
Shot 数据
```

都有了。

Workflow 可以创建：

```text
SHOT_GENERATION
```

注意：

Shot 生成通常还可以再拆成：

```text
Keyframe
 ↓
Video
```

---

# 17. 先生成 Shot1 Keyframe

Shot1：

```text
张默走向卧室门
```

Visual Worker 查询：

```text
shot_001
scene_001
char_001
asset_char_001
project.style
```

然后：

```text
Prompt Compiler
```

生成：

```text
cinematic realistic thriller,
night apartment corridor,
cold blue lighting,

same male character as reference image,
gray jacket,
black pants,

walking slowly toward bedroom door,

medium shot,
slow tracking camera,
vertical 9:16
```

然后：

```text
Image Provider
```

生成：

```text
shot_001_keyframe.png
```

上传 OSS。

数据库：

```text
assets
```

增加：

```text
SHOT_KEYFRAME
```

---

# 18. Shot1 Video Worker

现在 Shot1：

```text
status = KEYFRAME_READY
```

Workflow：

```text
创建 VIDEO_GENERATION job
```

↓

```text
video-queue
```

Video Worker 收到任务。

---

# 19. Video Worker 首先干什么

查询数据库：

```text
Shot1
Keyframe Asset
Character Asset
Scene
Project
```

然后：

```text
Prompt Compiler
```

得到动态 Prompt：

```text
The man slowly walks toward the bedroom door.

Natural walking motion.

Camera slowly tracks forward.

Maintain the exact character identity,
gray jacket,
cold nighttime lighting.

No sudden camera movement.
```

---

# 20. 调用真正的视频 API

VideoProvider：

```typescript
createTask({
    prompt,
    firstFrame: keyframe.url,
    duration: 4,
    ratio: "9:16"
})
```

如果采用 Runway，目前官方 API 的 image-to-video 就是：

```text
promptImage
+
promptText
+
model
+
ratio
+
duration
```

创建任务后会得到 task ID，再查询任务状态。

如果使用 Seedance，当前官方接口同样支持：

```text
文本
图片
视频
音频
```

等组合输入，图片还可以明确指定：

```text
role = first_frame
```

作为首帧条件。

---

# 21. 最关键的一点：此时并没有 video

Provider 返回：

```json
{
  "task_id": "provider_task_88721"
}
```

所以 Worker 此时不能写：

```text
shot.status = SUCCEEDED
```

应该：

```text
shot.status = GENERATING
```

Job：

```json
{
  "provider": "seedance",
  "provider_task_id": "provider_task_88721",
  "status": "RUNNING"
}
```

然后 Worker 就可以释放。

不应该：

```text
一个 Worker 傻等三分钟
```

---

# 22. 视频生成为什么需要异步

因为视频生成可能：

```text
几十秒
几分钟
甚至更久
```

当前万相官方 API 就明确采用：

```text
Create Task
 ↓
task_id
 ↓
Poll Result
```

的异步模式；任务成功后应把临时视频 URL 下载并保存到自己的 OSS。

Seedance 当前也是：

```text
POST 创建任务
 ↓
task ID
 ↓
GET 查询
```

或者直接使用 Webhook。

---

# 23. 谁负责知道任务完成？

两种方案。

### 方案 A：Polling

系统每隔：

```text
5～15 秒
```

查询：

```text
provider_task_88721
```

例如：

```text
queued
 ↓
running
 ↓
succeeded
```

---

### 方案 B：Webhook

我们给 Provider：

```text
callback_url
```

例如：

```text
https://api.xxx.com/webhooks/seedance
```

Provider 完成以后：

```http
POST /webhooks/seedance
```

告诉我们：

```json
{
  "id": "provider_task_88721",
  "status": "succeeded",
  "video_url": "https://provider.xxx/tmp.mp4"
}
```

Seedance 当前官方 API 支持 `callback_url`，状态包括 queued、running、succeeded、failed 和 expired。

生产环境通常更推荐：

```text
Webhook 为主
Polling 兜底
```

---

# 24. Shot1 视频成功以后，不能只保存 Provider URL

假设 Provider：

```text
video_url =
https://provider.com/temporary/xxx.mp4
```

这个 URL 很可能会过期。

例如 Seedance 当前查询接口说明生成视频 URL 有效期为 24 小时。

因此：

```text
Asset Worker
```

应该：

```text
下载 provider video
 ↓
校验文件
 ↓
上传自己的 OSS
```

最终：

```text
https://cdn.myapp.com/project_001/shot_001.mp4
```

数据库：

```json
{
  "asset_id": "asset_shot_001_video",
  "shot_id": "shot_001",
  "type": "VIDEO",
  "url": "https://cdn.myapp.com/..."
}
```

然后：

```text
shot_001.status = SUCCEEDED
```

---

# 25. 现在到了你之前最关心的地方

Shot2：

```text
张默伸手握住门把手
```

数据库：

```text
shot2.previous_shot_id = shot1
shot2.transition = continuous
```

Workflow 看到：

```text
continuous
```

于是决定：

```text
需要读取上一镜头结尾状态
```

---

# 26. Shot2 开始之前先读什么

Video Workflow：

```sql
SELECT *
FROM shots
WHERE id = 'shot_002'
```

得到：

```text
previous_shot_id = shot_001
```

再：

```text
查询 shot_001 VIDEO Asset
```

得到：

```text
shot1.mp4
```

---

# 27. 调用 extract_last_frame Tool

这里：

```text
Agent 不需要参与
```

Workflow 直接决定调用：

```text
extract_last_frame
```

底层：

```text
FFmpeg
```

例如：

```bash
ffmpeg \
-sseof -0.1 \
-i shot1.mp4 \
-vframes 1 \
last_frame.png
```

得到：

```text
last_frame.png
```

---

# 28. 什么时候存这张 last_frame

如果它：

```text
只用于本次立即生成
```

理论上可以临时保存。

但是正式产品建议：

```text
上传 OSS
```

因为它非常有价值：

```text
连续镜头恢复
重新生成
调试
质量分析
Prompt 回溯
用户编辑
```

所以 Asset：

```json
{
  "id": "asset_transition_001_002",
  "type": "IMAGE",
  "purpose": "TRANSITION_FRAME",
  "source_shot_id": "shot_001",
  "target_shot_id": "shot_002",
  "url": "https://..."
}
```

---

# 29. Shot2 真正的 Video Input

现在程序准备：

```text
firstFrame
```

：

```text
shot1 最后帧
```

Prompt：

```text
The man slowly raises his right hand
and grips the bedroom door handle.

Keep the same body position,
same clothes,
same apartment,
same lighting.

Close-up shot of the hand and door handle.
```

于是：

```typescript
videoProvider.createTask({
    firstFrame:
       transitionFrame.url,

    prompt:
       compiledPrompt,

    duration: 4,

    ratio: "9:16"
})
```

---

# 30. 这里系统实际上做了什么

不是：

```text
Shot1
+
Shot2
```

而是：

```text
Shot1 视频
      ↓
FFmpeg
      ↓
上一 Shot 视觉状态
      ↓
Shot2 初始状态
      +
Shot2 动作描述
      ↓
视频模型
      ↓
Shot2
```

也就是说：

```text
Image
告诉模型：
“从哪里开始”

Prompt
告诉模型：
“接下来做什么”
```

---

# 31. 甚至可以不自己 FFmpeg 抽尾帧

某些 Provider 已经支持：

```text
返回最后一帧
```

例如火山相关 Seedance 接口当前提供：

```text
return_last_frame
```

用于返回尾帧，可用于连续视频拼接。

于是：

```text
Shot1 createTask
```

的时候：

```json
{
  "return_last_frame": true
}
```

结果：

```text
video
+
last_frame
```

系统就可以直接保存：

```text
last_frame Asset
```

Shot2 直接使用。

这比：

```text
下载视频
→ FFmpeg 抽帧
```

更加高效。

---

# 32. Shot3 同理

Shot3：

```text
缓慢打开门
```

读取：

```text
Shot2 last frame
```

然后：

```text
Shot2 尾帧
+
Shot3 Prompt
```

生成：

```text
Shot3.mp4
```

---

# 33. Shot4 为什么不这么处理

Shot4：

```text
门后陌生人
```

Storyboard：

```text
transition = cut
```

说明镜头设计发生明显变化。

例如：

```text
Shot3：
男主背面

Shot4：
从男主肩膀后看房间内部
```

这时候：

```text
不能简单继承 Shot3 last frame
```

Workflow：

```typescript
if (transition === "continuous") {

    usePreviousLastFrame();

} else if (transition === "cut") {

    generateNewKeyframe();

}
```

所以 Shot4：

```text
重新生成 Keyframe
```

但是仍然读取：

```text
Character Reference

Scene Reference

Global Style
```

保证人物和世界观一致。

---

# 34. 四个 Shot 都完成以后发生什么

数据库：

```text
shot_001 SUCCEEDED
shot_002 SUCCEEDED
shot_003 SUCCEEDED
shot_004 SUCCEEDED
```

Workflow：

```text
检测 Episode 所有 Shot
```

如果：

```text
all succeeded
```

进入：

```text
QUALITY_REVIEW
```

而不是立即合成。

---

# 35. Review Worker 做什么

每个 Shot：

```text
视频
 ↓
抽取若干帧
 ↓
Vision Model
```

例如检查：

```text
人物是不是同一个人
衣服有没有改变
有没有多手
有没有严重畸变
动作是否符合 Prompt
画面有没有黑屏
与上一个 Shot 是否连续
```

返回：

```json
{
  "shot_id": "shot_002",
  "character_consistency": 0.93,
  "prompt_alignment": 0.88,
  "visual_quality": 0.91,
  "continuity": 0.86,
  "passed": true
}
```

---

# 36. 如果 Shot3 崩了怎么办

例如：

```json
{
  "visual_quality": 0.45,
  "passed": false,
  "reason": "右手出现额外手指"
}
```

Workflow：

```text
不是重新生成整部电影
```

只：

```text
shot3
```

进入：

```text
RETRYING
```

修改：

```text
retry_count = retry_count + 1
```

可能让 Prompt Compiler 加：

```text
natural human hands
correct anatomy
```

然后重新生成。

---

# 37. 这时候幂等开始非常重要

假设 Webhook 重复发了三次：

```text
shot3 succeeded
shot3 succeeded
shot3 succeeded
```

如果系统没幂等：

```text
可能触发 3 次下一阶段
```

于是需要：

```text
event idempotency
```

例如：

```text
provider_task_id
```

作为唯一键。

已经处理过：

```text
直接忽略
```

---

# 38. 对白音频先于正式分镜与视频

本节提前于视频阶段执行，章节编号保留历史导航意义，不表示调度顺序。台词确认后绑定项目级角色声音/逐句表演，先生成并核对真实音频、取得对齐时间；分镜依据实际时长，用户联合试听后才生成视频和画内口型。台词可跨 Shot，见 [音画规范](./dialogue-timeline.md)。

Script 里面可能：

```text
张默：
“谁在那里？”
```

Workflow：

```text
VOICE_GENERATION
```

Audio Worker 查询：

```text
character
dialogue
voice settings
```

调用：

```text
TTS Provider
```

得到：

```text
dialogue_001.wav
```

上传 OSS。

数据库：

```text
assets
```

：

```json
{
  "type": "AUDIO",
  "purpose": "DIALOGUE"
}
```

---

# 39. 为什么音频也必须记录 Asset

因为以后：

```text
重新剪辑
换字幕
调整音量
换画面
重新渲染
```

都不应该重新调用 TTS。

所以：

> 生成型服务的结果，只要未来可能复用，尽量资产化。

---

# 40. Subtitle 阶段

字幕使用确认文本与真实音频对齐，通过 AudioClip 映射全局时间，允许跨 Shot；最终组装阶段复用已有音频，不按每个镜头重复 TTS。下例的秒数仅用于说明，正式契约采用整数毫秒。

Dialogue：

```text
谁在那里？
```

Audio：

```text
2.1 秒
```

生成：

```text
subtitle segment
```

：

```json
{
  "text": "谁在那里？",
  "start": 14.3,
  "end": 16.4
}
```

最终生成：

```text
ASS
```

或者：

```text
SRT
```

短剧更建议 ASS。

因为可以：

```text
字体
位置
描边
动画
关键词高亮
```

---

# 41. Timeline 出现

正式模型分为视频生成前的 PerformancePlanVersion（音轨、镜头规划、画内外说话绑定）和生成完成后的 TimelineVersion（固定实际素材）。下例为历史简化成片示意，不代表对白必须为空或到此才开始音画规划，关系与字段以 [音画规范](./dialogue-timeline.md) 为准。

现在系统不是直接：

```text
ffmpeg shot1 shot2 shot3 shot4
```

最好先生成：

```text
Timeline
```

例如：

```json
{
  "video": [
    {
      "asset": "shot1.mp4",
      "start": 0,
      "duration": 4
    },
    {
      "asset": "shot2.mp4",
      "start": 4,
      "duration": 4
    },
    {
      "asset": "shot3.mp4",
      "start": 8,
      "duration": 5
    },
    {
      "asset": "shot4.mp4",
      "start": 13,
      "duration": 5
    }
  ],

  "dialogue": [],

  "music": [],

  "subtitle": []
}
```

Timeline 应该存数据库。

因为：

```text
它是“最终影片怎么组合”的工程文件。
```

---

# 42. Render Worker

Workflow：

```text
POST_PRODUCTION_READY
```

↓

创建：

```text
render job
```

↓

```text
render-queue
```

Render Worker：

```text
读取 Timeline
读取 Assets
```

然后执行：

```text
FFmpeg
```

或者：

```text
Remotion
```

---

# 43. FFmpeg 具体解决什么

它不负责理解剧情。

它负责：

```text
拼接视频

裁剪

转码

统一分辨率

统一 FPS

混音

加入 TTS

加入 BGM

字幕

淡入淡出

音量

导出 MP4
```

最终：

```text
final_v1.mp4
```

---

# 44. Final Video 再上传 OSS

Render Worker：

```text
生成本地文件
```

：

```text
/tmp/final.mp4
```

然后：

```text
uploadAsset()
```

最终：

```text
https://cdn.xxx.com/project_001/final_v1.mp4
```

数据库：

```json
{
  "asset_id": "final_001",
  "project_id": "project_001",
  "type": "VIDEO",
  "purpose": "FINAL"
}
```

Project：

```text
status = COMPLETED
```

---

# 45. 前端是怎么知道进度的

前端不应该：

```text
死循环轮询所有数据库
```

后端可以：

```text
WebSocket
```

推：

```json
{
  "projectId": "project_001",
  "stage": "VIDEO_GENERATION",
  "progress": 65
}
```

例如：

```text
Script      100%
Storyboard  100%
Characters  100%
Shot1       100%
Shot2       100%
Shot3        60%
Shot4         0%
Audio         0%
Render        0%
```

---

# 46. 到这里重新看一次整套运行链路

当前对白短剧执行链（其余旧段落仍以新规范为准）：

```text
用户
 ↓
Frontend
 ↓
POST /projects
 ↓
API Server
 ↓
保存 Project
 ↓
创建 Script Job
 ↓
BullMQ
 ↓
Script Worker
 ↓
Script Agent
 ↓
LLM
 ↓
保存 Script / Characters / Scenes
 ↓
用户确认剧本/台词
 ↓
角色声音/逐句表演 → TTS Worker → 音频归档/实际时长/可信对齐
 ↓
Workflow
 ↓
Storyboard Job
 ↓
Storyboard Worker
 ↓
Storyboard Agent
 ↓
LLM
 ↓
保存 Shots
 ↓
对白/分镜/画内外说话时间轴 → 用户联合试听确认
 ↓
Workflow
 ↓
Character Image Job
 ↓
Image Worker
 ↓
Image Provider
 ↓
OSS
 ↓
Assets
 ↓
Keyframe Job
 ↓
Keyframe Worker
 ↓
Image Provider
 ↓
OSS
 ↓
Video Job
 ↓
Video Worker
 ↓
Video Provider
 ↓
task_id
 ↓
Webhook / Polling
 ↓
Provider Video
 ↓
下载
 ↓
OSS
 ↓
Video Asset
 ↓
按 SpeechVisualBinding 执行画内口型（原生或独立适配）
 ↓
Review Worker
 ↓
Vision Model
 ↓
PASS / 显式恢复或重生成
 ↓
复用已确认对白音轨/字幕，校验跨镜头覆盖与口型
 ↓
Timeline
 ↓
Render Worker
 ↓
FFmpeg / Remotion
 ↓
final.mp4
 ↓
OSS
 ↓
Project COMPLETED
 ↓
前端播放
```

---

# 47. Redis、PostgreSQL、OSS 到底什么时候用

这是很多人最容易乱的地方。

## PostgreSQL

存：

```text
“事实”
```

例如：

```text
Project 是什么

Scene 是什么

Shot 是什么

Shot 当前是什么状态

生成任务 ID 是什么

Asset URL 是什么

失败几次

Timeline 是什么
```

服务器重启以后：

```text
这些东西不能丢
```

所以 PostgreSQL。

---

## Redis

存：

```text
“临时调度状态”
```

例如：

```text
BullMQ queue

Worker lock

并发控制

短期 cache

rate limiting
```

Redis 丢了：

```text
不能导致你的 Project 永久消失
```

---

## OSS / S3

存：

```text
二进制大文件
```

：

```text
PNG

JPG

MP4

WAV

MP3

ASS

SRT
```

---

# 48. 一个非常实用的数据原则

你可以记一句：

> **DB 存“它是什么”，OSS 存“它本身”。**

例如：

数据库：

```json
{
  "asset_id": "abc",
  "type": "VIDEO",
  "width": 1080,
  "height": 1920,
  "url": "https://..."
}
```

OSS：

```text
真正的 30MB mp4 文件
```

---

# 49. Tool Calling 到底什么时候用

不要把所有普通函数都叫 Agent Tool。

真正适合 Agent Tool Calling 的：

```text
search_reference
generate_script
generate_image
generate_video
generate_voice
review_video
```

因为 Agent 可能需要：

```text
根据当前内容决定是否调用
```

---

普通确定性操作：

```text
saveDatabase()

uploadOSS()

extractLastFrame()

updateStatus()

enqueueJob()
```

通常没有必要交给 LLM 自己决定。

Workflow 直接调用。

---

# 50. 所以系统里有两类 Tool

第一类：

```text
AI Tool
```

例如：

```text
LLMTool

ImageGenerationTool

VideoGenerationTool

TTSTool

VisionReviewTool
```

特点：

```text
不确定
成本高
结果不可完全预测
```

---

第二类：

```text
Engineering Tool
```

例如：

```text
FFmpeg

Storage

Database

Queue

HTTP

Image Processing
```

特点：

```text
确定性
代码控制
```

不要混淆。

---

# 51. Director Agent 到底干什么

到了这里你应该能看出来：

Director Agent 其实没有必要控制：

```text
数据库什么时候写

Redis什么时候入队

FFmpeg什么时候启动
```

Director 更适合：

```text
理解用户意图

决定故事结构

决定视觉策略

决定角色关系

决定哪些镜头应该连续

发现质量问题后决定如何调整
```

---

# 52. Workflow 干什么

Workflow：

```text
if script.done
   → storyboard

if storyboard.done
   → characters

if character.done
   → shots

if shot.failed
   → retry

if all shots.done
   → audio

if audio.done
   → render
```

非常机械。

这正是它应该做的事情。

---

# 53. Agent 与 Workflow 最简单的区分

以后看到任何功能就问：

```text
这件事有没有唯一确定答案？
```

如果：

```text
有
```

例如：

```text
Shot1 成功后保存数据库
```

用：

```text
Workflow / Code
```

如果：

```text
没有唯一答案
```

例如：

```text
这个剧情应该拆几个镜头？
```

用：

```text
Agent / LLM
```

---

# 54. Worker 为什么需要拆多个

推荐：

```text
script-worker

image-worker

video-worker

audio-worker

review-worker

render-worker
```

因为资源完全不同。

例如：

```text
Script Worker
耗 API Token

Video Worker
大量外部异步 API

Render Worker
吃 CPU

Image Worker
大量图片 IO
```

可以分别扩容。

---

# 55. 如果突然来了 100 个用户

假设：

```text
每个人
20 Shot
```

就是：

```text
2000 个视频任务
```

API Server 不应该：

```text
同时开 2000 个 Promise
```

而是：

```text
2000 jobs
     ↓
video-queue
     ↓
例如同时允许20个
     ↓
Video Worker Pool
```

这就是 Queue 的意义。

---

# 56. Provider 层为什么必须抽象

不要业务代码到处出现：

```typescript
callSeedance()
```

正确：

```typescript
interface VideoProvider {

    createTask(input)

    getTask(taskId)

    cancelTask(taskId)

}
```

然后：

```text
SeedanceProvider

RunwayProvider

WanProvider
```

Workflow 只认识：

```text
VideoProvider
```

---

# 57. Provider Router 以后甚至可以自动选模型

例如：

```text
人物近景
→ 模型 A

复杂运动
→ 模型 B

便宜预览
→ 模型 C

正式出片
→ 模型 D
```

这时候 Agent 可以参与：

```text
模型选择
```

但是 Provider 的 API 细节仍然由普通代码负责。

---

# 58. 系统真正核心的五类数据

最终你主要围绕：

```text
Project

Scene

Shot

Asset

Job
```

做开发。

Project：

```text
我要制作什么
```

Scene：

```text
故事发生在哪里
```

Shot：

```text
每一个生成单元
```

Asset：

```text
生成了什么东西
```

Job：

```text
现在正在执行什么
```

---

# 59. 最关键的是 Shot

整个视频系统的核心不是：

```text
Movie
```

而是：

```text
Shot
```

因为：

```text
生成

失败

重试

连续性

Prompt

关键帧

成本

质检
```

最终几乎都发生在：

```text
Shot
```

级别。

---

# 60. 一句话看懂一次 Shot 的完整生命周期

```text
PLANNED
 ↓
CONTEXT_READY
 ↓
KEYFRAME_GENERATING
 ↓
KEYFRAME_READY
 ↓
VIDEO_QUEUED
 ↓
VIDEO_GENERATING
 ↓
VIDEO_READY
 ↓
REVIEWING
 ↓
PASSED
```

失败：

```text
GENERATING
 ↓
FAILED
 ↓
RETRYING
 ↓
GENERATING
```

---

# 61. Shot2 的真实数据流可以压缩成这张“文字架构图”

```text
PostgreSQL
│
├── shot2
├── shot1
├── character
├── scene
└── assets
     │
     ▼
Video Worker
     │
     ├───────────────┐
     │               │
     ▼               ▼
Shot1 video       Character Ref
     │               │
     ▼               │
FFmpeg               │
     │               │
last_frame           │
     │               │
     └──────┬────────┘
            │
            ▼
      Prompt Compiler
            │
            ▼
     Video Provider
            │
            ▼
         task_id
            │
            ▼
   PostgreSQL Job
            │
            ▼
    Webhook / Poll
            │
            ▼
 Provider video URL
            │
            ▼
      Download Worker
            │
            ▼
          OSS
            │
            ▼
       Asset table
            │
            ▼
      shot2 = READY
```

这一段实际上就是整套系统最核心的缩影。

---

# 62. 推荐你的第一版实际代码结构

```text
apps/

  api/

  web/

workers/

  script-worker/

  image-worker/

  video-worker/

  audio-worker/

  review-worker/

  render-worker/

packages/

  agents/

     script-agent

     storyboard-agent

     review-agent

  workflow/

     project-workflow

     shot-workflow

  providers/

     llm/

     image/

     video/

     tts/

  tools/

     ffmpeg/

     storage/

     prompt-compiler/

  db/

  queue/

  schemas/
```

---

# 63. 第一版千万不要做得太复杂

首期剧情短剧闭环（Shot 数量由产品规格与 Provider 实测确定）：

```text
用户输入故事
 ↓
Script Agent
 ↓
确认剧本/台词 → 实际语音/对齐
 ↓
Storyboard Agent
 ↓
Shots / 对白时间轴 → 用户联合试听确认
 ↓
角色参考图
 ↓
每 Shot 一张 Keyframe
 ↓
图生视频
 ↓
画内口型同步 / 音画审核
 ↓
复用对白音轨/字幕、固定 Timeline
 ↓
FFmpeg
 ↓
MP4
```

第一版先不要：

```text
几十个 Agent

复杂长期 Memory

完全自主 Agent

复杂模型自动路由

高级 Timeline Editor

全自动配乐 Agent
```

---

# 64. 最后把整套系统压缩成一句工程公式

整套 Agent 文生视频平台其实就是：

```text
结构化业务数据
+
Agent 决策
+
确定性 Workflow
+
异步 Job Queue
+
AI Provider
+
Asset Pipeline
+
Media Rendering
```

再展开就是：

```text
用户需求
 ↓
结构化剧本
 ↓
确认台词 / 实际语音时长与对齐
 ↓
结构化分镜
 ↓
联合试听确认音画时间轴
 ↓
Shot
 ↓
生成任务
 ↓
Asset
 ↓
质检
 ↓
Timeline
 ↓
Render
 ↓
最终视频
```

如果你真正理解：

```text
Project
→ Scene
→ Shot
→ Job
→ Asset
→ Timeline
```

这条链，

基本就已经掌握了这类 **Agent 文生视频系统 70%～80% 的工程骨架**。

剩下的主要是：

```text
模型效果
Prompt Engineering
人物一致性
复杂 Workflow
性能
成本
生产稳定性
```

这些属于第二阶段。
