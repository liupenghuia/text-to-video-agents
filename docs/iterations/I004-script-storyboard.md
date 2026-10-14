# I004：剧本与分镜工作台

状态：Planned。前置为 I003 与核心音画路线的可行性结论；数据、时间与自然节奏规则见 [音画架构](../architecture/dialogue-timeline.md)。

- [ ] 实现 Script/Storyboard Agent、输出校验、角色形象/预设声音档案、普通话固定台词/逐句表演与两次确认。
- [ ] 实现 DialogueVersion/SpeechAsset/Alignment 与 Mock 实际音频，按时长规划 PerformancePlan/AudioClip/ShotPlacement/画内外 binding；实现文本核验与版本失效。
- [ ] 建立角色/场景/Shot 输入策略、显式依赖、快照及版本选择。
- [ ] 前端支持草稿、台词/角色试听、实际时长、跨镜头区间、联合确认、失败恢复与影响预览。

验收：无效输出不推进；一句跨两镜头/一镜头多句合法，仅画内需口型；联合试听能确认自然停顿、反应镜头、画内外续声与切点节奏，不为台词边界机械切镜；未联合确认不提交视频。
