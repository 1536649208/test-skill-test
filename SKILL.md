---
name: reference-searching
description: Search references of papers in a Zotero collection for papers related to a specific topic. Use when the user asks to find papers related to topic X from the references of papers in a Zotero collection/library, or wants to do citation chaining / reference mining from their Zotero library. Triggers include: "search references in my Zotero", "find papers about X from my library's references", "从我的Zotero库里找XX相关的参考文献", "在我的XX collection里搜参考文献".
---

# Reference Searching Skill

从指定的 Zotero collection 中，逐篇打开源文章的参考文献列表，按用户指定的主题过滤，输出匹配文献的 DOI 清单。

## Phase 0: 收集用户输入

每次启动时，必须明确以下 4 项：

1. **Zotero collection 名称** — 哪个 collection 存放源文章
2. **主题 / 特征关键词** — 参考文献需要匹配的主题（如"铝在电池热管理方面的应用"）
3. **输出目录** — 存放 `dois.txt` 和 `pending.txt` 的路径
4. **处理篇数** — 想处理多少篇源文章

如果用户在任何一项上未提供，主动询问。

## Workflow

### Phase 1: 发现源文章

1. 调用 `zotero_whoami` 确认 library 访问。
2. 调用 `zotero_list_collections` 找到目标 collection，获取其 `key`。
3. 调用 `zotero_search_items`，带上 `collectionKey`、`limit: 100`、`response_format: "concise"`，获取所有条目。**用 `concise` 而非 `detailed`**：此阶段只需要 key、title、itemType、collections、DOI、tags，`concise` 格式已包含全部这些字段，token 节省约 60%。（→ Pitfall 6）
4. **过滤 parent items**：只保留 `collections` 包含目标 collection key 且 `itemType` 为 `journalArticle`、`conferencePaper` 或 `book` 的条目。排除 `attachment` 和 `note` 子项。（→ Pitfall 1）
5. **CRITICAL — 检查已处理文章**：调用 `zotero_search_items`，`tag: "claude-read"`（`qmode: "everything"`），查出所有已被 `/reference-searching` 处理过的文章。`claude-read` 是全局标签，跨所有 collection 生效。排除已打此标签的文章。如果所有候选文章都已处理，报告并询问用户是否要重新处理某些。
6. **排序源文章**：综述（review）→ survey → 其余。主题关键词不出现在此阶段的优先级判断中。
7. 按用户指定的处理篇数截取列表，呈现给用户确认。已标记 `claude-read` 的文章显示为 "SKIPPED (已处理)"。

### Phase 2: 逐篇提取参考文献

对每篇选定的源文章，采用**关键词预筛 → 定向提取**的两步策略，避免拖回整段参考文献列表。

**第一步 — 检查 PDF 附件**：调用 `zotero_get_item`，`include_children: true`，查找 `itemType: "attachment"` 且 `contentType: "application/pdf"` 的子项。

**第二步 — 关键词预筛（TOKEN 优化的核心）**：

根据 Phase 0 用户输入的主题，系统化派生搜索关键词。**必须覆盖以下 4 个维度**：

| 维度 | 示例（主题："铝在电池热管理中的应用"） |
|------|------------------------------------------|
| 英文全称 | `aluminium`, `aluminum` |
| 元素符号 / 缩写 | `Al` (需配合上下文词如 `Al foam`, `Al heat sink`), `Al2O3`, `AlN` |
| 中文术语 | `铝`, `泡沫铝`, `铝合金` |
| 领域术语（可能隐含铝） | `metal foam`（电池热管理中 90% 是铝泡沫）, `porous metal` |

**派生后得到 6–8 个关键词**，用这些词作为 `query` 参数调用 `zotero_get_fulltext` 搜索 PDF 全文。

搜索参数：`max_passages: 3`、`max_chars: 3000`。

**如果无命中**：该论文不可能引用与主题相关的文献 → 直接跳过，不提取参考文献。

**如果有命中**：进入第三步。

**第三步 — 定向提取参考文献**：
- 调用 `zotero_get_fulltext`，`query: "References"`（或中文"参考文献"），定位参考文献段落。
- 使用 `max_passages: 4`、`max_chars: 8000`（从 20000 降至 8000——大多数参考文献列表在此范围内）。
- 如果参考文献段落被截断（`truncated: true`），用 `page_range` 补全缺失的页面。
- 解析提取到的文本，**重点关注包含预筛关键词的参考文献条目**，记录其标题和 DOI。

**第四步 — 内联标题过滤**：
- 提取参考文献时即时做标题过滤（Phase 3 Step 4），不要全部积累后再过滤。
- 仅保留标题中明显涉及用户主题概念的条目，其余直接丢弃。
- 只对保留下来的候选条目记录 DOI。

**无 PDF**：将该源文章记录到 `pending.txt`（格式：`文章名 — DOI`），跳过。

**关于 zotero_import**：不要调用 `zotero_import`。

### Phase 3: 两层过滤

#### Step 4 — 标题快速排除

从 Phase 2 提取到的参考文献中，仅看标题即可判断与主题无关的，直接排除。此步做粗筛。

#### Step 5 — 摘要深度判读（可选，仅在不确定时使用）

Phase 2 第四步已做内联标题过滤。对标题无法明确判断的**边缘文献**，才需要获取摘要：

获取摘要的手段（按优先级，优先使用免费结构化 API）：

1. **已在 Zotero 库中的文献**：调用 `zotero_get_fulltext` 读取摘要段落（`max_chars: 2000`）
2. **不在库中，有 DOI**：使用 **Crossref API** 直接获取结构化摘要（免费，无需 key）：
   ```
   WebFetch(url: "https://api.crossref.org/works/<DOI>", prompt: "提取 abstract 字段")
   ```
   返回纯 JSON，摘要字段为 `message.abstract`。一次调用，几十 token。
3. **不在库中，无 DOI 但有标题**：使用 **Semantic Scholar API**（免费，无需 key）：
   ```
   WebFetch(url: "https://api.semanticscholar.org/graph/v1/paper/search?query=<URL编码标题>&limit=1&fields=title,abstract", prompt: "提取 abstract 字段")
   ```
4. **兜底**：如果以上手段都无法获取摘要，保留该文献（宁可保留，不可误杀）

**Token 优化**：不逐篇获取摘要。仅对标题过滤后仍不确定的边缘文献（通常 < 20% 的候选条目）才进行摘要判读。标题明确相关的直接保留，标题明确不相关的已在 Phase 2 第四步丢弃。

判定时综合考虑：
- 标题和摘要是否直接涉及用户输入的主题概念
- 方法论或材料是否密切相关
- 宁可宽进，不可窄出 —— 让用户做最终决定

### Phase 4: 去重（CRITICAL — 不可跳过）

Phase 3 的匹配结果可能包含重复 —— 同一篇参考文献被多篇源文章引用时，会多次出现。写入前必须去重。

**去重键优先级**：

1. **DOI** — 首选。DOI 是文献唯一身份证，大小写不敏感（统一小写比较）。
2. **归一化标题** — 无 DOI 时使用。规则：lowercase → 去掉所有标点符号和多余空格 → 截断到前 80 字符。
3. **作者 + 年份 + 标题前缀** — 前两者都不满足时的兜底键。

**实现**：维护一个 `seen` 集合。对每条匹配文献：
- 计算去重键（先试 DOI，再试标题，最后组合键）
- 如果键已在 `seen` 中 → 跳过
- 否则加入 `seen`，保留该文献

**去重统计**：记录去重前后数量，在 Phase 7 报告中体现。

### Phase 5: 库内查重 — 排除已有文献（CRITICAL）

Phase 4 去重后的文献中，可能有些**已经在你的 Zotero 库里**。直接输出会造成冗余。

**查重策略（零额外 API 调用优先）**：

1. **源文章 DOI 比对**（免费，零 token）：
   - Phase 1 已拿到所有源文章的 DOI 列表
   - 如果候选 DOI 等于某篇源文章的 DOI → 直接归入 `dois_in_library.txt`
   - 这一步能排除大多数"已有"情况，且不产生任何额外 API 调用

2. **标题关键词搜索**（仅在候选 DOI 较少时使用，< 10 条）：
   - 对源文章比对后剩余的不确定 DOI
   - 从标题中提取 3–5 个最具区分度的词
   - 调用 `zotero_search_items(q="<关键词>", qmode="titleCreatorYear")` 确认是否在库中

**Token 优化**：优先使用策略 1（免费）。策略 2 仅在候选较少时使用；如果候选 > 10，直接全部标记为"未验证新增"，让用户在 `dois_new.txt` 中自行判断。

**输出文件**：

| 文件 | 内容 |
|------|------|
| `dois_new.txt` | 库中**没有**或**未验证**的 DOI —— 需要用户最终确认 |
| `dois_in_library.txt` | 库中**已有**的 DOI —— 已拥有，仅供参考 |
| `pending.txt` | 无法提取参考文献的源文章（同 Phase 2） |

### Phase 6: 输出匹配 DOI

将 Phase 5 分类后的 DOI 写入对应文件。

**写入前先确保输出目录存在**：
```bash
mkdir -p "<output_dir>"
```

### Phase 7: 标记源文章为已处理（CRITICAL — 不可跳过）

**每次运行结束后**，将所有本次处理的源文章打上 `claude-read` 标签（→ Pitfall 3）：

```text
zotero_manage_tags(action: "add", tags: ["claude-read"], item_keys: ["<key1>", "<key2>", ...])
```

关键点：
- 标记的是**源文章**（被提取参考文献的），不是找到的参考文献
- `claude-read` 是**全局标签**，跨所有 collection 生效
- 如果本次运行中用户明确要求重新处理某篇已标记文章，允许，但需标注"已处理过"
- 如果这一步被遗忘，下次运行会产生重复劳动

### Phase 8: 报告 & 自我改进

完成后呈现总结：

- 处理了多少篇源文章
- 提取了多少条参考文献
- 标题过滤排除了多少条
- 摘要判读后匹配了多少条
- **去重后保留多少条**（去除了多少条重复）
- **库内查重结果**：多少条是新增（不在库中），多少条已拥有
- 多少条 DOI 写入了 `dois_new.txt`
- 多少条 DOI 写入了 `dois_in_library.txt`
- 多少篇源文章因无 PDF 写入了 `pending.txt`
- 输出文件的完整路径

---

## Token 优化策略（CRITICAL — 每次运行必须遵守）

本 skill 的主要 token 消耗源及优化措施：

| 消耗源 | 优化前 | 优化后 | 节省 |
|--------|--------|--------|------|
| Phase 1 条目扫描 | `detailed` 格式 | `concise` 格式 | ~60% |
| Phase 2 参考文献提取 | 全文拖回 20000 chars | 关键词预筛→定向提取 8000 chars | ~70% |
| Phase 2 不相关论文 | 强制提取参考文献 | 关键词无命中直接跳过 | ~100% |
| Phase 3 摘要判读 | 逐篇获取摘要 | 仅边缘文献 | ~80% |
| Phase 5 库内查重 | 逐条 Zotero 搜索 | 源文章比对（免费）优先 | ~90% |

**核心原则**：先筛后取，宁可漏网不可拖全库。每次 API 调用前自问："这个调用真的需要吗？能不能用已有信息判断？"

---

## Self-Improvement 机制（CRITICAL）

**每次运行结束后**，review 本次运行中遇到的异常或错误：

1. 如果是**新类型的错误**（未出现在 Pitfall 列表中），在 Common Pitfalls 中新增一条
2. 如果是**已有 Pitfall 再次触发**，检查预防措施是否足够 —— 不够则加强措辞
3. 新增 Pitfall 格式：`### Pitfall N: <简短描述>`，然后 `**What happened**:`、`**Prevention**:`
4. 编号自动递增

目标：每次调用都比上一次更可靠。错误不是失败 —— 是让 skill 变强的数据。

---

## Common Pitfalls

### Pitfall 1: `zotero_search_items` 返回的条目中混入 attachment 和子项
**What happened**: `zotero_search_items` 返回了 96 条结果，实际只有 48 条是父条目 —— 一半是子附件。
**Prevention**: 始终按 `collections` 字段过滤。只有 `collections` 包含目标 collection key 的条目才是真正的父条目。`itemType: "attachment"` 且 `collections` 为空的条目是子项。

### Pitfall 2: 盲信 OpenAlex / 外部数据源的元数据
**What happened**: 部分参考文献的标题被截断、摘要缺失或作者信息不完整。盲信导致低质量过滤。
**Prevention**: 对于元数据不完整的边缘文献，不要直接丢弃。通过 WebSearch 补全信息后再做判断。

### Pitfall 3: 忘记在运行开始检查 `claude-read` 标签、运行结束打标签
**What happened**: 处理了源文章却没有先检查哪些已打过 `claude-read` 标签。也曾在结束后忘记打标签。前者导致重复劳动，后者导致下次运行没有记忆。
**Prevention**: Phase 1 Step 5 是强制的 —— 每次选源文章前必须查询 `tag: "claude-read"`。Phase 7 是强制的 —— 每次结束后必须给所有源文章打 `claude-read`。两步缺一不可。

### Pitfall 4: `dois.txt` 中出现重复 DOI
**What happened**: 同一篇参考文献被多篇源文章引用，在 Phase 3 匹配后多次出现，写入 `dois.txt` 时未去重，导致 DOI 索引中有重复条目。
**Prevention**: Phase 4 是强制去重步骤。以 DOI（优先）或归一化标题为键，维护全局 `seen` 集合。每次写入前必须经过去重。Phase 8 报告中必须报告去重数量。

### Pitfall 5: 找到的参考文献 DOI 用户库中已有
**What happened**: 参考文献挖掘找到的论文，实际上用户 Zotero 库里已经有了（甚至就在同一个 collection 中）。直接输出到 `dois.txt` 导致用户以为发现了新文献，实际是已知论文。
**Prevention**: Phase 5 是强制库内查重步骤。优先用源文章 DOI 比对（零 token），输出拆分为 `dois_new.txt`（新增/未验证）和 `dois_in_library.txt`（已拥有）。Phase 8 报告中分别报告新增和已有数量。

### Pitfall 6: Phase 2 全文拖回整个参考文献列表导致 token 爆炸
**What happened**: 每篇源文章用 `max_chars: 20000` 拖回整个参考文献段落。43 篇论文 × 20000 chars = 大量 token 消耗，其中大部分参考文献与主题无关。
**Prevention**: Phase 2 必须执行关键词预筛。先用用户主题关键词（如 `aluminium, Al2O3`）在 PDF 中搜索（`max_passages: 3, max_chars: 3000`）。无命中 → 跳过该论文。有命中 → 才提取参考文献段落（`max_chars: 8000`）。不相关的论文不产生参考文献提取成本。同时，Phase 1 使用 `concise` 格式、Phase 5 优先用源文章比对（免费）、Phase 3 摘要判读仅用于边缘文献。
