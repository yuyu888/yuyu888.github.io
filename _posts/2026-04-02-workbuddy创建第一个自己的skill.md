---
layout: mypost
title: workbuddy创建第一个自己的skill
categories: [AI]
---

ai发展一日千里， 使用workbuddy创建第一个自己的skill，记录下

以下内容也是用ai读取我的skill后生成

==================以下内容为ai生成====================

# 我的第一个 Skill

> 本文以一个真实的「考勤服务技能（ea\_attendance\_service）」为例，手把手带你了解 Skill 是什么、怎么创建，以及每个目录和文件的作用。

---

## 一、什么是 Skill？

在 WorkBuddy 中，**Skill（技能）** 是一个可复用的扩展包，它让 AI 助手具备某个领域的专业知识和可执行能力。

简单来说：

- **没有 Skill** 时，AI 只靠通用知识回答你的问题；
- **有了 Skill** 之后，AI 就像多了一位领域专家——它知道去哪里拿数据、用什么规则分析、输出什么格式的报告。

Skill 不只是一段提示词，它是 **知识 + 工具 + 工作流** 的组合体。

---

## 二、Skill 的目录结构

以 `ea_attendance_service` 为例，一个完整的 Skill 长这个样子：

```
ela_attendance_service/
├── SKILL.md                  ← 技能的"大脑"，必需
├── assets/
│   └── report_template.md    ← 报告模板等静态资产
├── references/
│   ├── api_documentation.md  ← 接口文档
│   └── attendance_rules.md   ← 业务规则说明
├── scripts/
│   ├── fetch_attendance_data.py   ← 数据获取脚本
│   ├── analyze_attendance.py      ← 数据分析脚本
│   └── generate_report.py         ← 报告生成脚本
└── reports/
    └── attendance_20260403_115117/  ← 历史生成报告（运行时产物）
```

下面逐个解释每个部分的职责。

---

## 三、各目录与文件详解

### 📄 SKILL.md —— 技能的核心配置

这是整个 Skill 最重要的文件，没有之一。它告诉 AI 助手：

- 这个技能叫什么名字、有什么用途；
- **什么情况下触发**（关键词匹配规则）；
- 数据源在哪、接口格式是什么；
- 业务逻辑（迟到/早退/加班的判断标准）；
- 每个脚本分别做什么事；
- 错误要怎么处理、缓存怎么管理。

文件开头是一段 YAML Front Matter，定义了元数据：

```yaml
---
name: "ea_attendance_service"
description: "考勤服务技能，用于查询和分析员工的考勤数据。"
---
```

**触发条件示例**：

```
当用户输入包含以下关键词时，自动使用此技能：
- "信息中心" + "考勤"
- "信息中心" + "打卡"
- "考勤信息"
- "打卡记录"
```

> 💡 **写 SKILL.md 的核心原则**：让 AI 读了就知道"我该做什么、怎么做、遇到问题如何处理"，越清晰越好。

---

### 📁 assets/ —— 静态资产目录

存放 Skill 运行时需要的**模板、配置、图表**等静态文件。

**`assets/report_template.md`** 是考勤报告的 Markdown 模板，定义了报告的标准结构：

```markdown
## 总体统计
- **统计人数**: [人数]人
- **出勤率**: [出勤率]%
- **迟到率**: [迟到率]%

## 员工排行
| 排名 | 员工姓名 | 加班天数 | 餐补金额 |
|------|----------|----------|----------|
```

脚本在生成报告时参照这个模板来填充实际数据，保证输出格式统一、专业。

> 💡 建议把所有"不会变化的内容骨架"放进 `assets/`，和动态数据分离。

---

### 📁 references/ —— 知识参考文档

这个目录存放 **让 AI 理解业务背景** 的文档，相当于技能的"内部知识库"。

#### `references/api_documentation.md` —— 接口文档

详细描述了数据源的调用方式：

- 数据 URL：`GET https://xxxx.com/permanent/document/20260403/attendance_all_list.json`
- 返回字段说明（`day`、`cname`、`first_checkin_time`、`last_checkin_time` 等）
- 过滤参数（`start_day`、`end_day`）
- 错误码对照表

```
| 错误码 | 说明     | 处理建议         |
|--------|----------|------------------|
| 0      | 成功     | 正常处理         |
| 2001   | 无数据   | 过滤后 data 为空 |
| 500    | 服务器错误 | 稍后重试        |
```

#### `references/attendance_rules.md` —— 考勤规则说明

这是 Skill 的"业务法规"，明确定义了所有判断标准：

| 规则类型 | 判断条件 | 备注 |
|----------|----------|------|
| 迟到 | 首次打卡 ≥ 10:00 | 仅工作日 |
| 早退 | 末次打卡 < 18:00 | 仅工作日 |
| 加班餐补 | 末次打卡 ≥ 20:00 | 仅工作日，每次 50 元 |
| 周末倒休 | `⌈在岗小时/2⌉ × 0.25` 天 | 周六日不计迟到/早退 |

> 💡 把规则写在 `references/` 而不是直接硬编码进脚本，好处是：规则变了只需改文档，AI 读文档就能理解最新口径，不需要改代码。

---

### 📁 scripts/ —— 可执行脚本目录

这是 Skill 的"双手"，真正执行数据处理工作的地方。三个脚本按职责分工，形成完整的处理流水线：

```
用户提问 → fetch（取数）→ analyze（分析）→ generate（报告）→ 输出结果
```

#### `scripts/fetch_attendance_data.py` —— 数据获取层

**职责**：从远端拉取全量考勤 JSON，按日期区间过滤，提供缓存。

核心设计亮点：

```python
class AttendanceDataFetcher:
    def __init__(self):
        self.cache_duration = 300  # 缓存 5 分钟，避免重复下载大文件

    def fetch_attendance_data(self, start_day, end_day):
        # 1. 先查本地缓存
        cached = self._load_from_cache(cache_key)
        if cached: return cached

        # 2. 缓存 miss → 下载全量快照
        all_records = self._fetch_source_records()

        # 3. 在内存里按日期过滤（YYYY-MM-DD 字符串比较）
        filtered = self._filter_by_day_range(all_records, start_day, end_day)

        # 4. 缓存结果供后续复用
        self._save_to_cache(cache_key, filtered)
        return filtered
```

**特别处理**：为避免中文乱码，优先用 UTF-8 解码响应体，失败则回退 GBK：

```python
for encoding in ("utf-8-sig", "utf-8", "gb18030", "gbk"):
    try:
        return json.loads(raw.decode(encoding))
    except: continue
```

---

#### `scripts/analyze_attendance.py` —— 数据分析层

**职责**：读取原始打卡记录，套用考勤规则，输出结构化的分析结果。

核心数据结构：

```python
@dataclass
class AttendanceRecord:
    date: str
    employee_name: str
    first_checkin: str
    last_checkin: str

    # 分析结果字段
    is_late: bool = False          # 是否迟到
    is_leave_early: bool = False   # 是否早退
    is_overtime: bool = False      # 是否加班（餐补口径）
    overtime_subsidy: float = 0.0  # 餐补金额
    is_weekend: bool = False       # 是否周末
    comp_off_days: float = 0.0     # 周末倒休天数

    def analyze(self):
        """判断工作日 / 周末，分别套用不同规则"""
        if 周末:
            倒休 = ceil(在岗小时 / 2) * 0.25
        else:
            判断迟到、早退、加班
```

分析器支持两种粒度：

- `analyze_employee(records, name)` → 个人维度分析
- `analyze_team(records)` → 团队维度分析，附加班排行榜

---

#### `scripts/generate_report.py` —— 报告输出层

**职责**：将分析结果格式化输出为 Markdown / JSON / CSV 文件。

```python
class AttendanceReportGenerator:
    def save_markdown_report(self, content, filename=None) → str
    def save_json_report(self, data, filename=None) → str
    def save_csv_summary(self, records, filename=None) → str
    def generate_daily_summary(self, date, records) → str    # 日报
    def generate_monthly_report(self, year_month, results) → str  # 月报
```

每次运行会在 `reports/` 目录下自动创建带时间戳的子文件夹，例如：

```
reports/attendance_20260403_115117/
├── attendance_report_20260403_115117.md
├── attendance_report_20260403_115117.json
└── attendance_summary_20260403_115117.csv
```

---

### 📁 reports/ —— 历史报告归档

运行时自动生成，不需要手工维护。每次查询都会在这里留下一份快照，方便后续追溯。

---

## 四、如何创建你自己的 Skill

### Step 1：确定 Skill 的职责边界

在动手之前，先明确三件事：

1. **触发场景**：用户说什么话时，这个 Skill 应该介入？
2. **数据来源**：数据从哪来？接口、文件、数据库还是本地文件？
3. **输出形式**：最终给用户看什么？纯文字、表格、Markdown 报告、图表？

### Step 2：创建目录结构

```bash
mkdir -p my_skill/{assets,references,scripts,reports}
touch my_skill/SKILL.md
```

### Step 3：写 SKILL.md（最关键的一步）

```markdown
---
name: "my_skill_name"
description: "一句话描述这个技能做什么"
---

# 技能名称

## 触发条件
当用户说 XXX 时触发...

## 数据源
- URL / 文件路径 / 接口说明

## 工作流程
1. 解析用户意图
2. 获取数据
3. 处理分析
4. 生成输出

## 错误处理
- 情况 A → 回复 XXX
- 情况 B → 回复 YYY
```

### Step 4：按需编写脚本

建议保持**职责分离**：

| 脚本 | 职责 |
|------|------|
| `fetch_xxx.py` | 只管拿数据，带缓存 |
| `analyze_xxx.py` | 只管分析，不碰 I/O |
| `generate_xxx.py` | 只管格式化输出 |

### Step 5：完善 references/ 文档

把业务规则、接口字段、边界情况统统写进 `references/`，让 AI 在分析数据时能随时查阅。

### Step 6：测试并安装

每个脚本都写一个 `main()` 测试函数，先独立跑通，再整体联调。

---

## 五、Skill 的整体工作流程回顾

```
用户说"信息中心帮我查一下上周的考勤"
         │
         ▼
   AI 读取 SKILL.md
   匹配到触发关键词 ✓
         │
         ▼
   fetch_attendance_data.py
   → 检查缓存 → 下载 JSON → 按日期过滤
         │
         ▼
   analyze_attendance.py
   → 解析记录 → 套用规则 → 计算迟到/加班/餐补
         │
         ▼
   generate_report.py
   → 生成 Markdown 报告 → 保存到 reports/
         │
         ▼
   AI 把报告内容展示给用户 ✅
```

---

## 六、几点经验总结

1. **SKILL.md 写得越详细，AI 表现越稳定。** 不要惜字如金，把你所有的业务知识都写进去。

2. **脚本职责要单一。** fetch / analyze / generate 三层分离，改一层不影响其他层，维护成本低。

3. **缓存是性能的关键。** 考勤快照文件可能很大，一定要做数据源缓存，避免每次查询都重复下载。

4. **references/ 让规则活起来。** 把考勤规则写在文档里，业务变了只需改文档，不用改代码。

5. **把边界情况写清楚。** 仅一次打卡、数据为空、周末上班……这些特殊情况都应该在 `references/attendance_rules.md` 里有明确说明。

---

> 现在，你已经完整了解了一个生产级 Skill 的全貌。去做你自己的第一个 Skill 吧！


