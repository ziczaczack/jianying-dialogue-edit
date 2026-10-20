---
name: jianying-dialogue-edit
description: Use when turning raw home/travel/vlog footage (DJI or phone clips, talkative or mostly-visual with little dialogue) into an edited 剪映 (JianYing) draft on this machine that the user opens in 剪映 to fine-tune.
---

# 剪映 对白驱动精剪流水线

## Overview

把几十分钟原始素材自动剪成几分钟精华版的 **可复用流水线**:不靠机械采样,而是用语音识别拿到对白时间戳 → 据内容选段 → 用 pyJianYingDraft 生成剪映草稿(转场+BGM)→ 注册进剪映索引让用户能打开微调。

**核心原则:对白是内容本身。先听人在说什么,再决定剪什么。** 素材对白稀少时,画面就是内容本身——见第 2 步的分支。

相关记忆:[[jianying-environment]] [[pyjianyingdraft-api-gotchas]] [[editing-preferences]] [[asr-faster-whisper-setup]] [[capcut-approach-decision]] [[dji-rotation-metadata]]

## 何时用

- 用户给一堆原始视频(家庭出游 / vlog),要剪成短片
- 需要"按重点剪"而不是简单拼接
- 产物要能在剪映里打开继续手动微调

不适用:已经有成熟时间线只差套模板;需要逐帧调色/合成(那是剪映里手工干的活)。

## 流水线(第 0 步 + 五步)

### 0. 素材体检(每批素材必做,产出一张表再往下走)
对每段跑 ffprobe,表里每段要有这四列:

| 列 | 怎么查 | 不正常时 |
|---|---|---|
| 时长 | `format=duration` | <1s 的碎片直接排除 |
| 音轨 | `-select_streams a` | 无音轨 = 延时/空镜,ASR 跳过,当纯画面素材 |
| 旋转 | `stream_side_data=rotation` | 非空就用 `-noautorotate` 抽一帧看原始像素:像素正 → 元数据错,`-display_rotation 0 -c copy` 去掉;像素侧躺 → 真竖拍,重编码烘焙,草稿里配模糊背景。修正版放 `<素材目录>\_rotfix\`,原片不动。详见 [[dji-rotation-metadata]] |
| 片段 id | DJI 文件名 `_` 分隔第 3 段(如 `0589`) | 跨天素材别用拍摄时刻截位当 id,会撞号 |

### 1. 转写全片对白(ASR)
faster-whisper(medium, GPU)把每段视频转成带时间戳文字稿。脚本 `scripts/transcribe_all.py [素材目录] [输出目录]`(带参数时用序号当 id、自动跳过无音轨段)。GPU 跑通的关键坑详见 [[asr-faster-whisper-setup]]。输出 `all.txt`(人读)+ 每段 json(机器读时间戳)。

### 2. 选高光段 —— 先数对白,再决定选段方式
读完 `all.txt`,数一下**有 ≥3 句像样对白的片段**占总片段数的比例:

- **≥ 一半 → 对白优先**:挑"有内容/有梗"的对白区间,跳过冷场、重复、纯闲聊;之后**抽查画面**(每段抽几帧)确认不是重复/失败镜头。
- **< 一半 → 画面优先**:每段每 2 秒抽一帧,拼成带「片段id + 秒数」标签的联系表,逐张看完再选段;纯画面段只取运镜到位、主体清楚的几秒;文字稿只用来给有对白的段卡句子边界。

两条路都要做:
- **地点/人物要交叉核对后才能写进字幕**:看招牌、地标,再按拍摄时间戳排顺序;不确定就不写地名。
- 产物 `KEEP = [(片段id, 起秒, 止秒, 速度, 原声音量), ...]`,按**故事线**排(到达→高潮→收尾)。多天旅行用"按天分章、章内按故事线"。

⚠️ **只看文字稿会犯的错(真实踩过):** 同一动作的多次尝试(起跳失败、重拍)在文字稿里都是"1 2 3",听着像不同高光,画面却几乎一样,接在一起就成了两段重复镜头夹一个无意义转场。

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

画面优先 / 多天素材的完整版见 `scripts/make_penang_draft.py`(每段带速度+音量、竖拍 `seg.add_background_filling("blur", 0.375)`、文字轨章节标题 + `TextBorder` 描边、BGM `add_fade` 首尾淡入淡出)。**背景填充、淡入淡出和转场一样,都要在 `add_segment` 之前加。**

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
| 没查旋转就进草稿 | 镜头在剪映里躺着放。第 0 步查 rotation + `-noautorotate` 看原始像素。 |
| 把一长串"啊""嗯"当对白 | 那是 Whisper 在噪音上的幻觉。对照画面,通常是空段。 |
| 凭印象写地名字幕 | 标错地点(把怡保写成乔治市)。招牌 + 时间戳交叉确认。 |
| 收进电脑屏幕/邮箱/OTP/付款画面 | 隐私外泄且拖节奏。默认排除,交付时说明。 |

## 产物交付
告诉用户:草稿名、时长、选了哪些高光(讲故事线)、排除了什么及原因、做过的素材修正(旋转等)、转场/BGM/字幕设置、需要用户做的事(退出剪映让我注册 / 看完给反馈)、以及 BGM 署名要求。
