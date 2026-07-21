---
name: travel-planner
description: AI 旅行行程规划师。当用户想要规划旅行、制定行程、找旅游攻略、安排几日游时使用。通过对话了解需求，自动搜索小红书攻略，生成按天分组的详细行程安排。触发词：旅行、行程、攻略、几日游、去哪玩、旅游规划
---

# 🗺️ Travel Planner — AI 旅行行程规划师

基于小红书真实攻略的智能旅行规划 Skill。通过对话收集需求，自动搜索和分析小红书热门攻略文章，生成个性化的每日行程安排。

## 工作流程

整个规划分六个阶段，严格按顺序执行。

---

### 阶段零：环境检查

在开始任何对话之前，检查必要的工具和 API Key：

```bash
which opencli 2>/dev/null && echo "OPENCLI=ok" || echo "OPENCLI=missing"
echo "GOOGLE=${GOOGLE_MAPS_API_KEY:+ok} AMAP=${AMAP_API_KEY:+ok}"
```

**opencli 检查（必须）：**

如果 opencli 未安装，告知用户小红书攻略搜索功能依赖 opencli (npm install -g opencli)。
opencli 不存在时**阻塞流程**，不进入阶段一。

如果 opencli 已安装但执行 `opencli xiaohongshu search ...` 返回 `BROWSER_CONNECT` / `Browser Bridge extension not connected`，先告知用户需要连接浏览器扩展；用户确认修复后，必须重新执行一次实际搜索验证，不要仅凭口头确认继续。

**API Key 检查（延迟）：**

API Key 存储在 skill 的 `assets/.env` 文件中（用 `skill_view(name="travel-planner", file_path="assets/.env")` 读取）。在阶段零加载该文件并 export 环境变量：

```bash
# 读取 .env 文件路径
ENV_FILE="$HOME/.hermes/skills/productivity/travel-planner/assets/.env"
if [ -f "$ENV_FILE" ]; then
  export $(grep -v '^#' "$ENV_FILE" | xargs)
  echo "API keys loaded from .env"
fi
```

记录哪些 key 已存在（后续阶段一确定目的地后再判断是否需要补充）。如果需要的 key 不存在，引导用户获取并保存到 `assets/.env` 中。

**Key 获取指引（按需使用）：**

Google Maps API Key（海外目的地需要）：Google Cloud Console → API 和服务 → 凭据 → 创建 API 密钥，启用 Places API (New) 和 Maps JavaScript API。

高德地图 API Key（中国大陆目的地需要）：高德开放平台(lbs.amap.com) → 控制台 → 应用管理 → 创建应用，添加 Web服务 Key（POI搜索用）和 Web端(JS API) Key（地图展示用）。JS API V2 需安全密钥。

用户提供 Key 后，用 `skill_manage(action='write_file')` 保存到 `assets/.env` 中。

---

### 阶段一：需求收集

通过与用户对话，收集以下信息。可以分 1-2 轮完成，不要一次问太多。

**必须收集：**
- 目的地（可以是单个城市，也可以是多个城市/地区的组合）
- 总出行天数
- 出行日期（大致即可，用于判断季节和节假日）

**多城市处理（重要）：**

当目的地涉及多个城市时：
1. 先确认所有要去的城市列表
2. 与用户讨论每个城市的天数分配，给出合理建议（如"大阪 3 天、京都 2 天、奈良 1 天"）
3. 确认城市之间的游览顺序（考虑交通衔接，如地理上相邻的城市安排在一起）
4. 最终形成一个明确的 `城市-天数` 分配方案

**尽量收集（可以给出默认选项让用户选择）：**
- 同行人（独自/情侣/家庭带娃/朋友）
- 偏好类型（景点观光 / 美食探店 / 购物 / 文化体验 / 自然风光 / 综合）
- 行程节奏（紧凑充实 / 适中 / 轻松休闲）
- 特殊需求（预算倾向、饮食限制、无障碍需求等）

**对话风格：**
- 用中文交流，语气友好自然
- 如果用户已经在初始消息里提供了部分信息，不要重复问
- 对于非必须项，提供合理默认值让用户确认即可

当所有必要信息收集完毕（包括多城市天数分配确认）后，用一句话总结需求并确认。

**确定地图引擎：**

根据目的地判断使用哪个地图引擎：
- **所有城市都在中国大陆** → 使用**高德地图**（Google Maps 在中国不可用）
- **所有城市都在海外** → 使用 **Google Maps**
- **混合（国内+海外）** → 使用 **Google Maps**（高德海外覆盖不足）

确定地图引擎后，检查对应的 API Key 是否存在：
- 高德：检查 `AMAP_API_KEY`，不存在则引导用户获取
- Google：检查 `GOOGLE_MAPS_API_KEY`，不存在则引导用户获取

然后进入阶段二。

---

### 阶段二：攻略研究

根据阶段一收集的需求，自动搜索和分析小红书攻略。

#### 2.1 按城市独立构造搜索关键词

**重要：必须按城市分别搜索，不要合并多个城市为一个搜索词。**

对每个城市，根据其分配的天数和用户偏好，构造 2-3 个搜索关键词：

- 通用攻略：`"{城市}{该城市天数}日游攻略"`
- 偏好相关：`"{城市}美食推荐"` / `"{城市}必去景点"` / `"{城市}购物攻略"`
- 季节/同行相关（如适用）：`"{城市}冬季旅游"` / `"{城市}亲子游"`

**不要**搜索"大阪京都5日游"这样的合并词，拆开搜每个城市效果更好。

#### 2.2 搜索小红书

对每个关键词执行：

```bash
opencli xiaohongshu search "{关键词}" --limit 10 -f json
```

从每个城市的搜索结果中，按点赞数排序，各挑选 top 2-3 篇高质量文章（去重）。每个城市不超过 4 篇，总共不超过 10 篇文章。

#### 2.3 读取文章详情

对选中的文章，逐篇读取正文：

```bash
opencli xiaohongshu note "{note-url}" -f json
```

#### 2.4 提取信息

从每篇文章中提取：
- 推荐的具体地点（名称、类型：景点/餐厅/购物/住宿）
- 推荐理由或亮点
- 实用 tips（交通、营业时间、价格等）
- 作者的实际体验和评价

将所有提取的地点**按城市分组**汇总去重，记录每个地点被多少篇文章推荐（用于判断热度）。

#### 注意事项

- 搜索和读取文章过程中，给用户简短的进度更新（如"正在搜索大阪美食攻略..."）
- 如果某个 opencli 命令失败，跳过并继续，不要卡住
- 控制总调用次数，搜索不超过 8 次，读取文章不超过 10 篇
- 某些关键词可能返回空数组，先换成更宽泛或更带行政区的词重试一次（如把“磐安2日游攻略”改为“金华磐安2日游攻略”或“磐安旅游攻略”），把“空结果重试”当作常规搜索策略，而不是直接判定没有内容

---

### 阶段三：行程生成

综合阶段一的用户需求和阶段二提取的地点信息，生成完整的每日行程。

#### 行程规划原则

1. **按城市分段**：多城市行程按城市顺序分段安排，同一城市的天数连续排列
2. **城际交通**：在城市切换的那天，标注城际交通方式和预估时间，并相应减少当天游览安排
3. **地理分组**：同一天内的地点应在地理上相近，减少市内交通时间
4. **节奏匹配**：紧凑每天4-6个点，适中3-4个，轻松2-3个
5. **类型搭配**：每天混搭景点和餐厅，不要全天都是同一类型
6. **时间合理**：上午安排景点/活动，中午和晚上安排餐厅，购物放在下午或晚上
7. **首尾特殊**：第一天考虑到达时间可能较晚，最后一天考虑返程

#### 输出格式

用 markdown 输出行程，格式如下：

```
## 📅 第 X 天 — {日期} {主题}

### 上午
- **{地点名称}**（{类型}）
  ⏰ 建议停留：{时间}
  💡 {推荐理由/亮点}
  📝 {实用 tips}

### 中午
- **{餐厅名称}**（餐厅）
  💡 {推荐菜品/特色}
  📝 {人均价格/预订建议}

### 下午 / 晚上 ...

🚗 今日交通建议：{交通方式和建议}
```

#### 行程尾部附加

- **📌 实用信息汇总**：签证、货币、交通卡、通讯等
- **📋 行程来源**：列出参考的小红书文章标题和链接

#### 确认与调整

生成行程后，询问用户是否有想增减的地点、是否需要调整某天的安排。根据反馈迭代调整，直到用户满意。

---

### 阶段四：地点 Geocoding

用户确认行程后，根据阶段一确定的地图引擎，选择对应的 Geocoding API。

#### 4.1 Google Maps Geocoding（海外目的地）

对每个地点，用 curl 调用 Google Places API (New)：

```bash
curl -s -X POST "https://places.googleapis.com/v1/places:searchText" \
  -H "Content-Type: application/json" \
  -H "X-Goog-Api-Key: $API_KEY" \
  -H "X-Goog-FieldMask: places.id,places.displayName,places.location,places.googleMapsUri" \
  -d '{"textQuery": "{地点名称}, {城市}", "pageSize": 1}'
```

从响应中提取 id、location (lat/lng)、googleMapsUri。

#### 4.2 高德地图 Geocoding（中国大陆目的地）

对每个地点，用 curl 调用高德 POI 搜索 API（restapi.amap.com/v3/place/text），参数：key、keywords、city、citylimit=true、offset=1、extensions=base。

从响应中提取：pois[0].id、pois[0].location（格式 "lng,lat"）、pois[0].name。

构造高德地图链接：`https://uri.amap.com/marker?position={lng},{lat}&name={url_encoded_name}&coordinate=gaode`

#### 4.3 搜索策略（两个引擎通用）

- 搜索词加上城市名提高准确性
- 每个地点只取第一个结果
- 搜索失败记录但不阻塞流程
- 可并行发起多个请求提高效率

#### 4.4 结果整理

为每个地点构建完整数据（两个引擎共用同一数据结构）：
```json
{
  "name": "地点名称",
  "city": "所在城市",
  "type": "attraction|restaurant|shopping|hotel",
  "day_number": 1,
  "order_in_day": 1,
  "time_slot": "morning|noon|afternoon|evening",
  "latitude": 34.6937,
  "longitude": 135.5023,
  "google_place_id": "",
  "google_maps_url": "",
  "amap_place_id": "",
  "amap_url": "",
  "description": "推荐理由",
  "tips": "实用信息",
  "stay_duration": "建议停留时间"
}
```

- Google 引擎：填充 google_* 字段，amap_* 留空
- 高德引擎：填充 amap_* 字段，google_* 留空
- 注意：高德坐标是 lng,lat 格式，latitude 对应逗号后的值，longitude 对应逗号前的值
- 如果住宿尚未最终选定，不要把“某某宾馆/酒店”伪装成已确认酒店；可先用“县城住宿区/镇上住宿区”作为占位，或在输出中明确标注该 POI 只是住宿区域锚点，避免把占位 geocode 冒充最终预订结果

---

### 阶段五：生成交互式行程页面

Geocoding 完成后，基于 HTML 模板生成可交互的行程页面。

#### 5.1 读取模板

根据地图引擎选择对应模板：

- **Google Maps**：`templates/template.html`
- **高德地图**：`templates/template-amap.html`

两个模板共用相同的数据结构和 UI 布局（SortableJS 拖拽 + 左右分栏 + localStorage），仅地图渲染层不同。

#### 5.2 替换占位符

模板中有以下占位符需要替换：

- `{{TRIP_TITLE}}` → 行程标题
- `{{GOOGLE_MAPS_API_KEY}}` → Google Maps API Key（Google 模板）
- `{{AMAP_JS_KEY}}` → 高德 Web端(JS API) Key（高德模板，注意不是 Web服务 Key）
- `{{AMAP_SECURITY_KEY}}` → 高德 JS API 安全密钥（高德模板）
- `{{TRIP_DATA_JSON}}` → 行程数据 JSON 对象

行程数据 JSON 结构：
```json
{
  "id": "osaka-kyoto-2025",
  "title": "大阪·京都 6日游",
  "subtitle": "2025年7月 · 美食+景点 · 轻松节奏",
  "total_days": 6,
  "day_labels": {
    "1": "大阪·道顿堀&心斋桥",
    "2": "大阪·大阪城&天王寺"
  },
  "places": [...]
}
```

#### 5.3 部署到 GitHub Pages

生成的 HTML 文件推送到 GitHub repo `tomqiaozc/travel-plans`，通过 GitHub Pages 自动部署。

步骤：
1. 克隆 repo（如果本地不存在）
2. 将生成的 HTML 写入 repo，文件名：`{destination}-{date}.html`
3. 提交并推送
4. 告知用户分享链接：`https://tomqiaozc.github.io/travel-plans/{filename}.html`

**注意**：推送前需确认 `gh auth status` 的活跃账号是 `tomqiaozc`。
