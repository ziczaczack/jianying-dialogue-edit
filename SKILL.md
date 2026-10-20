---
name: jianying-dialogue-edit
description: Use when turning raw home/travel/vlog footage into an edited 剪映 (JianYing) draft on this machine — selecting highlights, adding transitions and BGM, and generating a draft the user opens in 剪映. Covers ASR-driven highlight selection, pyJianYingDraft draft generation, and root_meta_info registration.
---

# 剪映 对白驱动精剪流水线

## Overview

把几十分钟原始素材自动剪成几分钟精华版的 **可复用流水线**:不靠机械采样,而是用语音识别拿到对白时间戳 → 据内容选段 → 用 pyJianYingDraft 生成剪映草稿(转场+BGM)→ 注册进剪映索引让用户能打开微调。

**核心原则:对白是内容本身。先听人在说什么,再决定剪什么。**

相关记忆:[[jianying-environment]] [[pyjianyingdraft-api-gotchas]] [[editing-preferences]] [[asr-faster-whisper-setup]] [[capcut-approach-decision]]

## 何时用

- 用户给一堆原始视频(家庭出游 / vlog),要剪成短片
- 需要"按重点剪"而不是简单拼接
- 产物要能在剪映里打开继续手动微调

不适用:已经有成熟时间线只差套模板;需要逐帧调色/合成(那是剪映里手工干的活)。

## 五步流水线

### 1. 转写全片对白(ASR)
faster-whisper(medium, GPU)把每段视频转成带时间戳文字稿。脚本见项目 `scripts/transcribe_all.py`。GPU 跑通的关键坑详见 [[asr-faster-whisper-setup]](把 `nvidia/*/bin/*.dll` 复制进 `ctranslate2/` 包目录;音频别用 Git Bash `/tmp`)。输出 `transcripts/all.txt`(人读)+ 每段 json(机器读时间戳)。

### 2. 通读文字稿,选高光段
人(我)读 `all.txt`,挑"有内容/有梗"的对白区间,跳过冷场、重复、纯闲聊。产物是 `KEEP = [(片段id, 起秒, 止秒), ...]`,按**故事线**排序(到达→过程高潮→收尾),不是按时间。

⚠️ **只看文字稿会犯的错(真实踩过):** 同一动作的多次尝试(比如全家起跳没成功、重拍)在文字稿里都是"1 2 3""一二三",听起来像不同高光,实则画面几乎一样。把它们接在一起 = 两段重复镜头中间一个无意义转场。**选段后必须抽查画面**(用 ffmpeg 抽帧或直接看),确认视觉上不是重复/失败镜头。

### 3. 定速度策略
- **移动镜头(走/跑/手持横移)绝不加速** —— 会让观众头晕。保持原速。
- 只对**定点/静止**的冗长片段考虑加速(volume=0 避免变调啸叫)。
- 详见 [[editing-preferences]]。

### 4. 生成剪映草稿(pyJianYingDraft)
参考 `scripts/make_dialogue_draft.py`。要点:

```python
# 1) 先建全部 VideoSegment(同一素材的 VideoMaterial 缓存复用)
src = Timerange(s0*1_000_000, (s1-s0)*1_000_000)
seg = VideoSegment(mat, Timerange(cursor, src.duration),
                   source_timerange=src, volume=1.0)   # 保留原声对白
cursor += src.duration

# 2) 转场必须在 add_segment 之前加(库在 add_segment 那刻收集转场素材)
for seg in segments[:-1]:
    seg.add_transition(TransitionType.叠化, duration=500_000)
for seg in segments:
    script.add_segment(seg)

# 3) BGM:新建 audio 轨,循环铺满全长,低音量垫底给对白让路
bgm = AudioMaterial(BGM_PATH)            # 如 Carefree.mp3
pos = 0
while pos < total:
    d = min(bgm.duration, total - pos)
    script.add_segment(AudioSegment(bgm, Timerange(pos, d),
                       source_timerange=Timerange(0, d), volume=0.18))
    pos += d
```

更多 API 坑(变速 target 自动覆盖、中文枚举转场名)见 [[pyjianyingdraft-api-gotchas]]。

### 5. 注册进剪映索引
剪映靠 `root_meta_info.json` 的 `all_draft_store` 发现草稿;外部生成的草稿必须手动插入一条。**改前备份 `.bak`;改前确认剪映已完全退出**,否则剪映关闭时会用内存里的旧索引覆盖你的改动。

## BGM 选择与版权
- 关键词搜场景情绪:欢乐休闲 / 治愈 / 悲伤;或上抖音找热门二创音频接轨。
- 用 CC-BY 免费音乐(如 Kevin MacLeod)时,**发布需署名**:`Music: "曲名" by 作者 (来源链接), CC BY`。
- 搞笑处(被吓、出糗)可点缀梗音效制造笑点。

## Common Mistakes

| 错误 | 后果 / 纠正 |
|---|---|
| 只读文字稿就选段 | 接到重复/失败镜头。**抽查画面**再定稿。 |
| 移动镜头加速 | 观众头晕。只对定点片段加速。 |
| 转场在 `add_segment` 之后加 | JSON 里转场为空。必须**先加转场再 add_segment**。 |
| 剪映开着时改 `root_meta_info.json` | 注册被覆盖,草稿不显示。先彻底退出剪映。 |
| 加速片段不静音 | 变调啸叫。`volume=0`。 |
| ASR 用 `/tmp` 路径 | Windows 原生 Python 找不到。用项目内真实路径。 |
| 按时间顺序排片段 | 平淡。按**故事线**排(起承转合)。 |

## 产物交付
告诉用户:草稿名、时长、选了哪些高光(讲故事线)、转场/BGM 设置、需要他做的事(退出剪映让我注册 / 看完给反馈)、以及 BGM 署名要求。
