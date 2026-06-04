# 赛博出气筒 MVP 完整开发路线图

> 技术栈：Next.js 14 + TypeScript + DeepSeek API + Canvas 2D
> 预计工期：6-8周（Vibe Coding辅助）
> 目标：跑通"捏脸→情景再现→结案报告"完整闭环

---

## 总览：五个阶段

```
阶段一：环境搭建与技术验证        第1周
阶段二：捏脸系统                  第2-3周
阶段三：情景再现（核心）          第3-5周
阶段四：预设剧本与结案报告        第5-6周
阶段五：留存机制与上线准备        第7-8周
```

---

## 阶段一：环境搭建与技术验证（第1周）

### 1.1 项目初始化

```bash
npx create-next-app@latest cyber-punching-bag \
  --typescript --tailwind --app --src-dir
cd cyber-punching-bag
npm install
```

**目录结构：**
```
src/
├── app/
│   ├── page.tsx              # 首页/剧本选择
│   ├── build/page.tsx        # 捏脸页
│   ├── battle/page.tsx       # 对战页（情景再现）
│   └── report/page.tsx       # 结案报告
├── components/
│   ├── PixelAvatar.tsx       # 像素捏脸组件
│   ├── BattleChat.tsx        # 对战对话组件
│   └── Advisor.tsx           # 军师组件
├── lib/
│   ├── deepseek.ts           # AI调用封装
│   ├── prompts.ts            # 所有prompt管理
│   └── presets.ts            # 预设剧本数据
└── types/
    └── index.ts              # 类型定义
```

### 1.2 DeepSeek API 接入验证

**这是第一个必须验证的技术节点，不通则后续全部作废。**

先写一个最简单的测试脚本，不涉及任何UI：

```typescript
// src/lib/deepseek.ts
const DEEPSEEK_BASE = "https://api.deepseek.com/v1"

export async function chat(
  messages: { role: string; content: string }[],
  model: "deepseek-chat" | "deepseek-reasoner" = "deepseek-chat"
) {
  const res = await fetch(`${DEEPSEEK_BASE}/chat/completions`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${process.env.DEEPSEEK_API_KEY}`,
    },
    body: JSON.stringify({
      model,
      messages,
      stream: true,          // 必须开流式，否则等待时间太长用户会以为卡死
      max_tokens: 300,
      temperature: 0.9,
    }),
  })
  return res                  // 返回原始Response，由调用方处理stream
}
```

**环境变量配置：**
```bash
# .env.local
DEEPSEEK_API_KEY=your_key_here
```

**⚠️ 难点提示：流式输出（Streaming）**

这是整个项目最容易踩坑的技术点之一。DeepSeek返回的是Server-Sent Events格式，需要在Next.js的Route Handler里正确处理，否则用户看到的是整段文字突然出现，没有打字机效果。

```typescript
// src/app/api/villain/route.ts
import { NextRequest } from "next/server"
import { chat } from "@/lib/deepseek"

export async function POST(req: NextRequest) {
  const { messages, systemPrompt } = await req.json()
  
  const response = await chat([
    { role: "system", content: systemPrompt },
    ...messages
  ])

  // 关键：把DeepSeek的stream直接透传给前端
  return new Response(response.body, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache",
      "Connection": "keep-alive",
    },
  })
}
```

前端用`EventSource`或手动读取`ReadableStream`消费这个流，每收到一个chunk就追加到对话气泡里。

**验收标准：** 在浏览器里看到反派AI逐字打出一句阴阳怪气的话，本阶段完成。

---

### 1.3 双AI并发调用验证

军师AI需要在反派AI说完之后，立刻分析并给出回怼建议。这涉及两个AI的串联调用。

```typescript
// 调用顺序：
// 1. 用户发言 → 反派AI响应（stream）
// 2. 反派AI响应完毕 → 触发军师AI分析（stream）
// 3. 两个结果分别渲染在屏幕不同位置

async function handleUserInput(userMessage: string, conversationHistory: Message[]) {
  // Step 1: 反派AI
  const villainResponse = await fetchVillainResponse(userMessage, conversationHistory)
  
  // Step 2: 等反派说完，军师同步分析
  // 注意：军师不需要等反派stream结束，可以在反派说到一半时就开始分析
  // 但MVP阶段先串行，避免复杂度
  const advisorResponse = await fetchAdvisorResponse(villainResponse, conversationHistory)
  
  return { villainResponse, advisorResponse }
}
```

**⚠️ 难点提示：状态管理**

对战页面有三个独立的状态流：
1. 对话历史（用于构建context）
2. 反派AI当前streaming状态
3. 军师AI当前streaming状态

建议用`useReducer`而非多个`useState`，否则状态更新顺序会出问题。

---

## 阶段二：像素捏脸系统（第2-3周）

### 2.1 素材设计（开发前必须完成）

**这是最容易被忽视但最关键的前置工作。**

素材质量决定产品调性，代码写得再好，素材难看就完了。

**推荐方案：外包给像素画师**

在闲鱼/即时设计社区/Fiverr发需求：
```
需求：像素风格人物零件素材包
风格：FC/GB游戏风格，黑白线稿+有限色板（建议4-8色）
尺寸：每个零件 64x64px，PNG透明背景
数量：
  - 脸型基底：4种（圆/方/胖/瘦）
  - 发型：8种（地中海/大背头/三根毛/油腻分头等）
  - 眼睛：8种（死鱼眼/势利眼/小眯眯眼/铜铃眼等）
  - 嘴巴：6种（厚唇/歪嘴/假笑/撇嘴等）
  - 特征贴纸：8种（黑痣/油汗/黑眼圈/眼镜/胡子等）
  - 服装：4种（西装/格子衬衫/廉价polo/土味夹克）
预算：300-800元
```

**如果预算有限，用AI生成素材的正确姿势：**

用Midjourney或DALL-E，prompt模板：
```
pixel art, game boy style, 64x64, transparent background, 
[具体器官描述], limited 4-color palette, black outline, 
white background, flat design, no shading
--style raw --ar 1:1
```

关键是**同一批素材用同一个seed**，保证风格一致。

### 2.2 Canvas图层合成实现

```typescript
// src/components/PixelAvatar.tsx
"use client"
import { useEffect, useRef } from "react"

interface AvatarConfig {
  face: string      // 图片路径，如 "/sprites/face/round.png"
  hair: string
  eyes: string
  mouth: string
  details: string[] // 可多选，如黑痣+眼镜
  outfit: string
}

export function PixelAvatar({ config, size = 128 }: { 
  config: AvatarConfig
  size?: number 
}) {
  const canvasRef = useRef<HTMLCanvasElement>(null)

  useEffect(() => {
    const canvas = canvasRef.current
    if (!canvas) return
    const ctx = canvas.getContext("2d")
    if (!ctx) return

    // 关键设置：禁用抗锯齿，保持像素锐利感
    ctx.imageSmoothingEnabled = false
    
    // 清空画布
    ctx.clearRect(0, 0, size, size)

    // 按图层顺序绘制（顺序很重要）
    const layers = [
      config.face,
      config.outfit,
      config.hair,
      config.eyes,
      config.mouth,
      ...config.details,
    ].filter(Boolean)

    // 异步加载所有图层后一次性渲染
    Promise.all(
      layers.map(src => loadImage(src))
    ).then(images => {
      ctx.clearRect(0, 0, size, size)
      images.forEach(img => {
        ctx.drawImage(img, 0, 0, size, size)
      })
    })
  }, [config, size])

  return (
    <canvas
      ref={canvasRef}
      width={size}
      height={size}
      style={{ imageRendering: "pixelated" }} // CSS层面也禁用模糊
    />
  )
}

function loadImage(src: string): Promise<HTMLImageElement> {
  return new Promise((resolve, reject) => {
    const img = new Image()
    img.onload = () => resolve(img)
    img.onerror = reject
    img.src = src
  })
}
```

**⚠️ 难点提示：`imageRendering: pixelated`**

这行CSS至关重要。如果不加，浏览器会对Canvas内容做平滑处理，像素风就毁了。在不同浏览器里这个属性的写法略有差异，需要加兼容：

```css
image-rendering: pixelated;        /* Chrome, Safari */
image-rendering: crisp-edges;      /* Firefox */
image-rendering: -moz-crisp-edges; /* 旧Firefox */
```

### 2.3 捏脸UI交互

捏脸页面的交互逻辑：左侧选择面板，右侧实时预览。

```typescript
// 捏脸状态管理
const [avatarConfig, setAvatarConfig] = useState<AvatarConfig>({
  face: "/sprites/face/round.png",
  hair: "/sprites/hair/bald.png",   // 默认地中海
  eyes: "/sprites/eyes/dead.png",
  mouth: "/sprites/mouth/sneer.png",
  details: [],
  outfit: "/sprites/outfit/suit.png",
})

// 每次用户点击零件，立刻更新预览
function updatePart(part: keyof AvatarConfig, value: string) {
  setAvatarConfig(prev => ({ ...prev, [part]: value }))
}
```

**同时收集文字特征输入（喂给AI，不影响图像）：**

```typescript
const [personality, setPersonality] = useState({
  name: "",           // 代号，如"王总"
  traits: [],         // 多选标签：爱画大饼/道德绑架/阴阳怪气
  customDetail: "",   // 自由输入："说话喜欢拖长音，开会爱抖腿"
  relation: "",       // 直属上级/隔壁组同事/亲戚
})
```

**验收标准：** 点击不同零件，右侧像素头像实时更新，视觉流畅无闪烁。

---

## 阶段三：情景再现核心系统（第3-5周）

### 3.1 Prompt工程（最重要的非代码工作）

**这是整个产品体验质量的天花板，比任何代码都重要。**

在写任何界面之前，先把prompt调到满意。建议用ChatGPT/Claude的对话界面反复测试，不要急着写代码。

**反派AI System Prompt模板：**

```
你是一个叫「{name}」的{relation}，正在扮演一个让用户深感受气的角色。

【你的人设】
{traits_description}
用户描述的你的特点：{customDetail}

【本次场景】
{scene_description}
用户想重演的那句话：{trigger_sentence}

【你的行为规则】
1. 用第一人称说话，像真人一样，不要解释自己在"扮演"
2. 每次只说1-3句，留给用户回应的空间
3. 擅长的话术：画大饼、道德绑架、偷换概念、阴阳怪气，根据人设选择
4. 当用户回怼有力时，你可以表现出短暂的语塞，但很快找补
5. 绝对不要主动认错或道歉，除非用户的回击达到5次且非常有力
6. 语言风格要口语化、真实，带点职场/家庭的具体细节感
7. 严禁涉及政治、色情、真实人身威胁等内容

【血条机制】
用户的每次回应会被军师评分（1-10分）。当你累计承受30分时，触发「清算结局」：
你开始结巴、找借口、最终狼狈收场。这是用户应得的胜利时刻。
```

**军师AI System Prompt模板：**

```
你是用户的高情商军师，正在观看一场对话。

【刚才反派说的话】
{villain_last_message}

【对话上下文】
{conversation_history}

你需要完成两件事：

【第一件：话术识别】
用一句话点破反派用的是什么套路，要犀利、要准、要让用户恍然大悟。
格式：「⚠️ 他在用【话术名称】：一句话解释」
例：「⚠️ 他在用【道德绑架】：把公司利益和你的个人价值绑在一起，让你不好意思拒绝」

【第二件：三档回击】
给出三个回击选项，用户可以直接用或参考：
🕊️ 温和版（保住关系）：[一句话]
⚔️ 犀利版（不卑不亢）：[一句话]  
💣 核武器版（掀桌子）：[一句话]

要求：
- 每句话不超过30字
- 要像真人说话，不要书面语
- 核武器版可以稍微损一点，但不要人身攻击
- 给这次用户回应评分（1-10），用于血条计算，格式：「评分：X」（放在最后，用户不可见）
```

**⚠️ 难点提示：Prompt的迭代**

第一版prompt跑出来的效果大概率不满意。建议：
- 准备10个典型场景，每个场景跑5轮对话
- 记录哪些地方"出戏"（反派突然道歉、军师分析太书面、回击建议太弱）
- 针对性修改prompt，重新测试
- 至少迭代3-5轮才能达到可上线水准

这个工作不能省，是产品的灵魂。

### 3.2 场景输入页面

```typescript
// 结构化填空组件
const SCENE_FIELDS = [
  {
    id: "location",
    label: "这件事发生在",
    type: "select",
    options: ["会议室", "工位旁边", "微信/钉钉上", "领导办公室", "餐桌上", "电话里"],
  },
  {
    id: "trigger",
    label: "让你最憋屈的一句话",
    type: "text",
    placeholder: "尽量原话，越准确越爽",
  },
  {
    id: "reaction",
    label: "你当时的反应",
    type: "select", 
    options: ["没说话，忍了", "敷衍过去", "说了但没发挥好", "当场怼了但没怼赢"],
  },
  {
    id: "extra",
    label: "还有什么细节",
    type: "text",
    placeholder: "选填，比如当时还有谁在场、之前的背景",
  },
]
```

**进入对战的过渡动画（重要的仪式感）：**

用户点击"开始清算"后，不要直接跳转，做一个3秒的像素风过场：
```
画面：像素小人走进一个房间
文字逐字出现："正在还原案发现场..."
然后反派像素头像从屏幕右侧走入
```

这个细节会显著提升代入感，用户会更投入。

### 3.3 对战界面布局

```
┌─────────────────────────────────┐
│  🩸 王总的嚣张值  ████████░░ 80% │  ← 血条，顶部常驻
├─────────────────────────────────┤
│                                 │
│  [王总像素头像]                  │
│  ┌─────────────────────────┐    │
│  │ 你这个项目做成这样，     │    │  ← 反派气泡，左对齐
│  │ 让我怎么跟老板交代？     │    │
│  └─────────────────────────┘    │
│                                 │
│  ┌─────────────────────────┐    │
│  │ 你当时说了：            │    │  ← 用户历史发言，右对齐
│  │ "对不起王总..."         │    │
│  └─────────────────────────┘    │
│                                 │
├─────────────────────────────────┤
│ 军师悄悄说：                    │  ← 军师面板，可折叠
│ ⚠️ 他在用【甩锅话术】           │
│ 🕊️ "这个问题我们一起复盘下"    │
│ ⚔️ "需求变更了3次，我记录在案"  │
│ 💣 "王总您当时审核通过的"       │
├─────────────────────────────────┤
│ [输入框] 这次你想怎么说？  [发送]│  ← 底部输入
└─────────────────────────────────┘
```

**⚠️ 难点提示：移动端键盘弹出问题**

在手机上，输入框获得焦点时软键盘弹出，会把整个布局顶上去，导致对话内容被遮挡。这是移动端Web开发最常见的坑之一。

解决方案：
```typescript
useEffect(() => {
  // 监听viewport变化（键盘弹出/收起）
  const handleResize = () => {
    // 键盘弹出时，滚动到最新消息
    messagesEndRef.current?.scrollIntoView({ behavior: "smooth" })
  }
  window.visualViewport?.addEventListener("resize", handleResize)
  return () => window.visualViewport?.removeEventListener("resize", handleResize)
}, [])
```

### 3.4 血条系统实现

```typescript
// 血条状态
const [villainHP, setVillainHP] = useState(100)
const [totalScore, setTotalScore] = useState(0)

// 从军师AI返回的文本里提取评分
function extractScore(advisorText: string): number {
  const match = advisorText.match(/评分：(\d+)/)
  return match ? parseInt(match[1]) : 3  // 默认3分
}

// 每轮对话结束后更新血条
function updateHP(score: number) {
  const damage = score * 2  // 满分10分=扣20HP，5轮清算完成
  setTotalScore(prev => prev + score)
  setVillainHP(prev => Math.max(0, prev - damage))
  
  // 血条归零时触发清算结局
  if (villainHP - damage <= 0) {
    triggerFinalScene()
  }
}
```

**血条扣减动画（像素风格）：**
血条用CSS实现，扣减时做一个短暂的红色闪烁+数字跳动，配合8-bit音效（可用`tone.js`或直接用`AudioContext`生成方波音）。

### 3.5 清算结局场景

这是整个产品最高潮的时刻，要做得足够爽：

```typescript
function triggerFinalScene() {
  // 1. 反派像素头像开始抖动动画
  // 2. 血条爆裂特效（像素爆炸粒子）
  // 3. 反派说最后一段求饶/狼狈的话（专门的prompt）
  // 4. 屏幕出现像素风 "清算完成！" 大字
  // 5. 自动跳转结案报告页
}
```

清算结局的反派prompt：
```
用户已经完成了清算。现在你需要表现出彻底的溃败：
结巴、找借口失败、最终承认自己理亏。
但要符合人设，不要过于夸张，要有现实感。
这是用户应得的心理闭合时刻。
```

---

## 阶段四：预设剧本与结案报告（第5-6周）

### 4.1 预设剧本数据结构

```typescript
// src/lib/presets.ts
export interface PresetScript {
  id: string
  npc: {
    name: string
    role: string
    avatar: AvatarConfig      // 预设的像素头像配置
    personality: string[]     // 话术标签
    voiceStyle: string        // 说话风格描述，给prompt用
  }
  scene: {
    title: string             // 显示给用户的标题
    location: string
    background: string        // 场景背景描述
    openingLine: string       // 反派开场白
    difficulty: "初级" | "中级" | "地狱"
  }
  tags: string[]              // 如 ["职场", "PUA", "加班"]
}

export const PRESET_SCRIPTS: PresetScript[] = [
  {
    id: "boss-meeting",
    npc: {
      name: "王总",
      role: "直属上级",
      avatar: { face: "round", hair: "bald", eyes: "small", mouth: "sneer", details: ["sweat"], outfit: "suit" },
      personality: ["画大饼", "道德绑架", "阴阳怪气"],
      voiceStyle: "说话喜欢拖长音，经常用'我们'代替'你'，擅长把公司利益和个人价值绑定",
    },
    scene: {
      title: "早会，王总当众点名批评你",
      location: "会议室",
      background: "周一早会，全组10人在场，王总看着你上周的周报皱眉头",
      openingLine: "小[姓]啊，你这个数据……唉，我不说了，你自己看看，这叫什么东西？",
      difficulty: "初级",
    },
    tags: ["职场", "当众批评", "PUA"],
  },
  {
    id: "colleague-meetingroom",
    npc: {
      name: "小李",
      role: "同级同事",
      avatar: { face: "thin", hair: "neat", eyes: "sly", mouth: "smile", details: ["glasses"], outfit: "casual" },
      personality: ["甩锅", "背刺", "当面一套背后一套"],
      voiceStyle: "表面热情，说话喜欢加'你知道的'、'大家都清楚'，擅长在关键时刻切割责任",
    },
    scene: {
      title: "会议上，小李把锅甩给你",
      location: "会议室",
      background: "季度复盘会议，项目出了问题，领导正在追责",
      openingLine: "王总，这块其实当时我跟[你姓]说过的，让他注意一下，你知道的，他可能当时比较忙……",
      difficulty: "中级",
    },
    tags: ["职场", "甩锅", "背刺"],
  },
  {
    id: "auntie-newyear",
    npc: {
      name: "二姑",
      role: "亲戚",
      avatar: { face: "round", hair: "perm", eyes: "sharp", mouth: "wide", details: ["mole"], outfit: "festive" },
      personality: ["催婚催生", "比较贬低", "关心即控制"],
      voiceStyle: "说话喜欢用'为你好'开头，擅长举邻居家孩子的例子，对方越沉默越来劲",
    },
    scene: {
      title: "年夜饭，二姑开始催婚",
      location: "餐桌",
      background: "大年三十，全家吃饭，二姑喝了点酒，开始关心你的终身大事",
      openingLine: "哎，[你名字]，你今年多大了？隔壁翠花家的孩子比你小两岁，孩子都会跑了！",
      difficulty: "中级",
    },
    tags: ["家庭", "催婚", "亲戚"],
  },
]
```

### 4.2 结案报告生成

结案报告是产品的传播引擎，设计要像游戏结算界面。

```typescript
// 生成结案报告的prompt
const REPORT_PROMPT = `
根据以下对话记录，生成一份「情绪结案报告」。

【对话记录】
{conversation_history}

【要求】
生成一个JSON，包含以下字段：
{
  "verdict": "一句定性的话，判定这场交锋的胜负和性质",  // 如："完胜！成功识破连环道德绑架"
  "tactics_detected": ["话术1", "话术2"],                 // 识别到的话术，2-4个
  "best_comeback": "你说过的最强一句话",                  // 从对话里挑
  "score": 85,                                            // 综合评分 0-100
  "rank": "初出茅庐",                                     // 段位，从低到高：初出茅庐/后来居上/见招拆招/炉火纯青/出口成章
  "next_tip": "下次遇到这种情况，还可以这样说：[一句话建议]"
}
只返回JSON，不要其他内容。
`
```

**报告页面视觉设计（像素风游戏结算界面）：**

```
┌──────────────────────────────┐
│  ⚔️  情绪结案报告  ⚔️         │
│  ══════════════════════      │
│                              │
│  判决：完胜！                │
│  成功识破连环道德绑架         │
│                              │
│  识别话术：                  │
│  [道德绑架] [偷换概念]        │
│                              │
│  本场最强回击：               │
│  "需求变更了3次，记录在案"    │
│                              │
│  综合评分                    │
│  ████████████░░  85分        │
│                              │
│  段位：见招拆招               │
│                              │
│  [📸 截图分享] [再战一局]     │
└──────────────────────────────┘
```

**⚠️ 难点提示：截图分享**

网页里实现"截图并分享"需要用`html2canvas`库，把DOM元素转成图片：

```bash
npm install html2canvas
```

```typescript
import html2canvas from "html2canvas"

async function shareReport() {
  const reportEl = document.getElementById("report-card")
  if (!reportEl) return
  
  const canvas = await html2canvas(reportEl, {
    backgroundColor: "#1a1a2e",  // 像素风深色背景
    scale: 2,                    // 2倍分辨率，截图更清晰
  })
  
  // 转成图片下载
  const link = document.createElement("a")
  link.download = "赛博出气筒_结案报告.png"
  link.href = canvas.toDataURL()
  link.click()
}
```

注意：`html2canvas`对自定义字体支持不好，报告卡片里的字体要用系统字体或提前加载好。

---

## 阶段五：留存机制与上线准备（第7-8周）

### 5.1 宿敌主动推送（Web Push Notification）

```typescript
// 申请推送权限
async function requestPushPermission() {
  const permission = await Notification.requestPermission()
  if (permission === "granted") {
    const registration = await navigator.serviceWorker.ready
    const subscription = await registration.pushManager.subscribe({
      userVisibleOnly: true,
      applicationServerKey: process.env.NEXT_PUBLIC_VAPID_KEY,
    })
    // 把subscription发给后端保存
    await savePushSubscription(subscription)
  }
}
```

**推送文案（反派视角，定时触发）：**
- 距离上次清算已经24小时：`"怎么，昨天那句话还没想好怎么回？"`
- 周五下午5点：`"小X啊，这周的周报写完了吗？我等着看呢。"`
- 周一早上9点：`"新的一周，来热热身？"`

**⚠️ 注意：** Web Push在iOS Safari上直到iOS 16.4才支持，且需要用户先把网页添加到主屏幕。如果主要用户是iPhone，这个功能效果会打折，可以降级为"每日邮件提醒"作为替代。

### 5.2 进度持久化

用户捏的宿敌和对话进度需要持久化，否则刷新就丢了。

MVP阶段用`localStorage`即可，不需要后端数据库：

```typescript
// 保存宿敌档案
function saveVillain(villain: VillainProfile) {
  localStorage.setItem(`villain_${villain.id}`, JSON.stringify(villain))
}

// 读取所有宿敌
function loadAllVillains(): VillainProfile[] {
  const keys = Object.keys(localStorage).filter(k => k.startsWith("villain_"))
  return keys.map(k => JSON.parse(localStorage.getItem(k) || "{}"))
}
```

后期如果要做跨设备同步，再接后端（Supabase是最快的选择）。

### 5.3 上线前检查清单

**功能检查：**
- [ ] 捏脸→情景输入→对战→结案报告全流程跑通
- [ ] 预设3个剧本全部可以正常进入和完成
- [ ] 流式输出在手机Chrome/Safari上正常显示
- [ ] 截图分享功能生成的图片在微信里可以正常预览
- [ ] 血条归零触发清算结局

**性能检查：**
- [ ] 首屏加载时间 < 3秒（Vercel自动CDN，一般没问题）
- [ ] 像素素材全部压缩（TinyPNG处理，单张 < 50KB）
- [ ] API调用加loading状态，避免用户以为卡死

**内容安全：**
- [ ] 所有prompt里加入安全围栏
- [ ] 前端输入框加基础敏感词过滤
- [ ] 结果里出现明显问题内容时有fallback处理

**部署：**
```bash
# Vercel一键部署
npm install -g vercel
vercel deploy
```

---

## 关键里程碑与验收节点

| 时间 | 里程碑 | 验收标准 |
|------|--------|----------|
| 第1周末 | API联调完成 | 在终端里看到双AI流式输出 |
| 第3周末 | 捏脸完成 | 点击零件，像素头像实时更新，无闪烁 |
| 第5周末 | 对战核心完成 | 完整跑通一场对战，血条正常扣减 |
| 第6周末 | 预设剧本+结案报告 | 6个剧本全部可玩，报告可截图 |
| 第7周末 | 留存机制 | 推送权限可申请，宿敌档案可保存 |
| 第8周末 | 上线 | Vercel部署，手机端体验流畅 |

---

## 技术难点汇总（避坑优先级排序）

| 优先级 | 难点 | 解决方案 |
|--------|------|----------|
| 🔴 最高 | Prompt质量 | 上线前至少迭代5轮，每轮跑10个场景 |
| 🔴 最高 | 流式输出在Next.js的实现 | 用Route Handler透传stream，前端ReadableStream消费 |
| 🟡 中 | 移动端键盘弹出布局错乱 | 监听visualViewport resize事件 |
| 🟡 中 | Canvas像素模糊 | imageRendering: pixelated + imageSmoothingEnabled: false |
| 🟡 中 | html2canvas字体问题 | 结案报告卡片只用系统字体 |
| 🟢 低 | iOS Web Push兼容 | 降级为邮件提醒 |
| 🟢 低 | localStorage跨设备 | MVP阶段不需要，后期接Supabase |

---

## 推荐的Vibe Coding工作流

1. **每个功能模块单独开一个对话**，不要在一个超长上下文里做所有事
2. **先写类型定义**（TypeScript interface），让AI理解数据结构再写逻辑
3. **遇到AI生成的代码跑不通**，把报错信息完整粘贴回去，不要自己猜
4. **Canvas和音频相关代码**，让AI写完后重点人工检查，这类代码AI经常生成看起来对但细节有bug的版本
5. **Prompt调试**建议在Claude或ChatGPT的对话界面做，而不是在代码里调，效率更高
