# Story Package v1 规范

> 版本：v0.1 | 状态：初版定稿

## 1. 概述

Story Package 是一个可游玩故事的完整内容单元。它包含世界设定、角色定义、主线结构、Prompt 模板等一切 AI 推演所需的内容。

设计原则：
- 一个 Story Package = 一个完整的可游玩故事
- 与引擎代码完全解耦，纯数据驱动
- 同一引擎可加载不同 Story Package
- 开发态为明文目录，发布态可打包加密

## 2. 目录结构

```
story-packages/
  {story-id}/                      # 故事 ID，kebab-case
    package.json                   # 包元信息
    world.md                       # 世界设定文档
    rules.json                     # 世界规则（AI 硬约束）
    characters/
      {character-id}.json          # 每个角色一个文件
    plot/
      outline.json                 # 主线大纲
      chapters/
        chapter-{N}.json           # 每章详情
    player/
      roles.json                   # 可选的玩家角色列表
    prompts/
      system.md                    # System Prompt 模板
      narrator-style.md            # 旁白风格模板
      character-injection.md       # 角色注入模板
      milestone-steering.md        # 里程碑引导模板
      output-format.md             # 输出格式指令模板
```

## 3. package.json

```json
{
  "id": "three-kingdoms-chibi",
  "version": "1.0.0",
  "schemaVersion": 1,
  "title": "三国演义 · 赤壁弧线",
  "subtitle": "从隆中对到火烧赤壁",
  "description": "玩家扮演刘备麾下谋士，亲历三顾茅庐到赤壁之战的波澜壮阔。",
  "author": "Studio Name",
  "tags": ["historical", "strategy", "politics", "chinese-classic"],
  "language": "zh-CN",
  "estimatedPlaytime": "2-4h",
  "difficulty": "medium",
  "playerRoles": ["advisor"],
  "chapterCount": 5,
  "coverImage": "assets/cover.webp",
  "minimumEngineVersion": "0.1.0"
}
```

### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 唯一标识，kebab-case |
| `version` | semver | 是 | 内容版本号 |
| `schemaVersion` | number | 是 | Story Package 规范版本 |
| `title` | string | 是 | 显示标题 |
| `subtitle` | string | 否 | 副标题 |
| `description` | string | 是 | 简介（200 字以内） |
| `author` | string | 是 | 作者/工作室 |
| `tags` | string[] | 是 | 标签，用于筛选和推荐 |
| `language` | string | 是 | BCP 47 语言标签 |
| `estimatedPlaytime` | string | 否 | 预计游玩时长 |
| `difficulty` | enum | 否 | easy / medium / hard |
| `playerRoles` | string[] | 是 | 可选玩家角色 ID 列表 |
| `chapterCount` | number | 是 | 章节总数 |
| `coverImage` | string | 否 | 封面图路径 |
| `minimumEngineVersion` | semver | 是 | 要求的最低引擎版本 |

## 4. world.md

Markdown 格式的世界设定文档。供 AI System Prompt 引用。

```markdown
# [世界名称]

## 时代背景
[时代、地理、社会结构概述]

## 核心冲突
[世界中的根本矛盾和驱动力]

## 势力格局
[主要势力、联盟、对抗关系]

## 重要地点
[关键场景简述]

## 历史时间线
[故事开始前的重要事件]
```

## 5. rules.json

世界规则约束，AI 必须遵守的硬性限制。

```json
{
  "allowed": [
    "冷兵器战斗",
    "骑马通信",
    "中医治疗",
    "天象占卜"
  ],
  "forbidden": [
    "现代科技（枪械、电话、电力）",
    "魔法或超自然能力",
    "穿越时空",
    "打破第四面墙"
  ],
  "communicationSpeed": "骑马传信，城市间需要数日",
  "technologyLevel": "东汉末年，公元 208 年前后",
  "socialRules": [
    "严格的君臣尊卑关系",
    "以字号称呼表示尊重",
    "女性角色受时代礼教约束"
  ],
  "customRules": []
}
```

## 6. characters/{character-id}.json

```json
{
  "id": "cao-cao",
  "name": "曹操",
  "courtesy_name": "孟德",
  "title": "丞相",
  "faction": "cao-wei",
  "isPlayerSelectable": false,

  "personality": {
    "core_traits": ["ambitious", "cunning", "decisive"],
    "big_five": {
      "openness": 0.8,
      "conscientiousness": 0.7,
      "extraversion": 0.6,
      "agreeableness": 0.3,
      "neuroticism": 0.4
    },
    "motivation": "统一天下，结束乱世",
    "fear": "被部下背叛，功业未成"
  },

  "voice": {
    "tone": "威严中带戏谑，喜怒不形于色",
    "speech_pattern": "善用反问和试探，话中有话",
    "vocabulary_level": "文雅但不迂腐，偶尔引用诗文",
    "catchphrases": ["有趣", "说下去"],
    "forbidden_expressions": ["求你了", "我不知道该怎么办"],
    "emotion_spectrum": {
      "calm": "语气从容，带微笑，话少但字字有重量",
      "angry": "声音压低而非提高，用冷酷的事实威胁",
      "pleased": "大笑，慷慨许诺，但心思难测",
      "suspicious": "突然沉默，目光如炬，连续反问"
    }
  },

  "relationships": {
    "liu-bei": { "initial": -20, "description": "政治对手，暗中忌惮其仁德之名" },
    "zhuge-liang": { "initial": 0, "description": "尚未交手，但听闻其才" },
    "player": { "initial": 0, "description": "视为刘备阵营的无名谋士" }
  },

  "knowledge_boundary": {
    "knows": ["己方军力部署", "北方政局", "刘备近况概要"],
    "does_not_know": ["诸葛亮的具体计策", "东吴的真实态度", "玩家的真实身份"]
  },

  "dialogue_examples": [
    {
      "context": "初次见面，试探对方",
      "line": "久闻刘皇叔麾下多有能人，今日一见，倒想听听阁下高见。"
    },
    {
      "context": "被激怒但压制",
      "line": "好一个伶牙俐齿。不过……空口白牙可退不了百万雄兵。"
    }
  ]
}
```

### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 角色唯一 ID |
| `name` | string | 是 | 角色名 |
| `faction` | string | 否 | 所属势力 ID |
| `isPlayerSelectable` | bool | 是 | 玩家是否可以扮演此角色 |
| `personality.core_traits` | string[] | 是 | 2-4 个核心性格关键词 |
| `personality.big_five` | object | 否 | 大五人格维度 (0.0-1.0) |
| `personality.motivation` | string | 是 | 行为驱动力 |
| `personality.fear` | string | 是 | 最大恐惧 |
| `voice` | object | 是 | 说话风格定义 |
| `voice.emotion_spectrum` | object | 是 | 不同情绪下的说话方式 |
| `relationships` | object | 是 | 与其他角色的初始关系 |
| `knowledge_boundary` | object | 是 | 角色的知识边界 |
| `dialogue_examples` | array | 是 | 代表性对话（至少 2 条） |

## 7. plot/outline.json

```json
{
  "storyId": "three-kingdoms-chibi",
  "chapters": [
    {
      "chapter": 1,
      "title": "三顾茅庐",
      "summary": "刘备三访隆中，玩家作为随行谋士亲历此过程。",
      "scenes": ["longzhong-village", "zhuge-cottage"],
      "activeCharacters": ["liu-bei", "zhuge-liang", "guan-yu", "zhang-fei"],
      "milestones": ["ms_longzhong_plan"],
      "savePointsGranted": 3
    }
  ],
  "milestones": [
    {
      "id": "ms_longzhong_plan",
      "chapter": 1,
      "title": "隆中对",
      "summary": "诸葛亮提出三分天下的战略构想",
      "mandatory": true,
      "prerequisites": [],
      "triggerCondition": {
        "type": "compound",
        "operator": "and",
        "conditions": [
          { "type": "scene", "sceneId": "zhuge-cottage" },
          { "type": "flag", "flag": "liu_bei_visited_three_times", "value": true }
        ]
      },
      "effects": {
        "flags": { "longzhong_plan_delivered": true },
        "relationships": { "zhuge-liang->liu-bei": 30, "zhuge-liang->player": 10 },
        "unlockChapter": 2
      },
      "narrativeDirective": "这是一个庄严的时刻。诸葛亮应展现远见卓识，刘备应表现出诚恳和激动。叙事风格应庄重而充满希望。"
    }
  ]
}
```

### 里程碑触发条件类型

| type | 参数 | 说明 |
|------|------|------|
| `scene` | `sceneId` | 玩家处于指定场景 |
| `flag` | `flag`, `value` | 指定 flag 达到指定值 |
| `relationship` | `pair`, `min`/`max` | 关系值达到阈值 |
| `turn_count` | `min` | 至少经过 N 轮 |
| `compound` | `operator`, `conditions` | 组合条件 (and/or) |

## 8. plot/chapters/chapter-{N}.json

```json
{
  "chapter": 1,
  "title": "三顾茅庐",
  "openingNarrative": "建安十二年冬，刘备率关羽、张飞并一众随从，第三次来到隆中……",
  "scenes": [
    {
      "id": "longzhong-village",
      "name": "隆中村口",
      "description": "山间小村，竹林环绕，薄雾缭绕。",
      "atmosphere": ["serene", "anticipation"],
      "activeCharacters": ["liu-bei", "guan-yu", "zhang-fei"],
      "beats": [
        {
          "id": "ch1_s1_b1",
          "type": "narration",
          "contentDirective": "描写隆中村口的景色，渲染刘备第三次到访的坚定与忐忑。",
          "interventionType": "none"
        },
        {
          "id": "ch1_s1_b2",
          "type": "dialogue",
          "speaker": "zhang-fei",
          "contentDirective": "张飞抱怨：一个村夫何须三请？展现张飞的急躁和不满。",
          "interventionType": "reactive",
          "relatedVariables": ["zhang_fei_patience"]
        },
        {
          "id": "ch1_s1_b3",
          "type": "dialogue",
          "speaker": "liu-bei",
          "contentDirective": "刘备训斥张飞，强调诚意的重要性。转向玩家询问意见。",
          "interventionType": "forced"
        }
      ]
    }
  ]
}
```

### Beat 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | Beat 唯一 ID |
| `type` | enum | 是 | narration / dialogue / event / transition |
| `speaker` | string | dialogue 时必填 | 发言角色 ID |
| `contentDirective` | string | 是 | 给 AI 的生成指令（不是最终文本） |
| `interventionType` | enum | 是 | none / reactive / forced |
| `relatedVariables` | string[] | 否 | 此 Beat 可能影响的变量 |

## 9. player/roles.json

```json
{
  "roles": [
    {
      "id": "advisor",
      "name": "无名谋士",
      "description": "刘备麾下新招募的年轻谋士，身份神秘，才华初显。",
      "defaultRelationships": {
        "liu-bei": 30,
        "guan-yu": 10,
        "zhang-fei": 5,
        "zhuge-liang": 0,
        "cao-cao": 0
      },
      "customizable": {
        "displayName": true,
        "backstory": false
      }
    }
  ]
}
```

## 10. prompts/ 目录

所有 Prompt 模板使用 Markdown 编写，通过模板变量 `{{variable}}` 注入运行时数据。

### system.md 示例

```markdown
你是一个文本游戏的 AI 叙事引擎。你的职责是在以下世界设定中扮演所有 NPC 并推进故事。

## 世界设定
{{world_summary}}

## 世界规则
{{world_rules}}

## 当前场景
{{current_scene}}

## 叙事风格
{{narrator_style}}

## 重要约束
- 严格遵守世界规则，不生成违反设定的内容
- 每个角色的对话必须符合其角色声音卡定义
- 输出必须是指定的 JSON 格式
```

### 模板变量列表

| 变量 | 来源 | 说明 |
|------|------|------|
| `{{world_summary}}` | world.md 摘要 | 世界设定概要 |
| `{{world_rules}}` | rules.json | 格式化的世界规则 |
| `{{current_scene}}` | 当前 chapter/scene | 场景描述 |
| `{{narrator_style}}` | narrator-style.md | 旁白风格指令 |
| `{{active_characters}}` | 运行时 | 当前活跃角色的 Profile |
| `{{memory_context}}` | 运行时 | 分层记忆内容 |
| `{{plot_context}}` | 运行时 | 当前里程碑和进度 |
| `{{player_action}}` | 运行时 | 玩家本轮输入 |

## 11. 校验规则

Story Package 加载时需校验：

- `package.json` 所有必填字段存在且类型正确
- 所有角色 ID 在 `characters/` 下有对应文件
- `outline.json` 中引用的角色 ID 都存在
- 里程碑的 `prerequisites` 不形成循环依赖
- Beat 引用的 `speaker` 是该场景的 `activeCharacters` 之一
- 章节编号连续，无跳跃
- Prompt 模板中的 `{{variable}}` 都有对应的运行时数据源
