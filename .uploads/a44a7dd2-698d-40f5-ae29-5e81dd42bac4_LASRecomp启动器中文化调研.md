# LASRecomp-Android-Plus 启动器中文化 · 阶段调研报告

| 项目 | 内容 |
|---|---|
| 仓库 | [Nycolazs/LASRecomp-Android-Plus](https://github.com/Nycolazs/LASRecomp-Android-Plus) |
| 调研 commit | `cca791a3e7218e1dac95e7d4a11b8f97dba3ea51`（2026-10-23） |
| 调研日期 | 2026-11-09 |
| 调研方式 | 本地浅克隆后逐文件核对（`git clone --depth 1 --filter=blob:none --no-checkout`） |
| 需求范围 | **仅翻译启动器（App 外壳 UI）**；游戏内中文为游戏自身内置，不在本次范围内 |

---

## 1. 结论速览

1. **i18n 不在 App 层，而在运行时 UI 层**：`runtime/ui/` 下四个文件。
2. 项目已有完整的多语言骨架（EN + PT-BR + ES/FR/DE/IT/NL），**加语言是"加一条线"，不是从零搭 i18n**。
3. 翻译机制是**惰性翻译**：界面代码里所有文案保持英文，绘制那一刻才经 `T()` 查表。
4. 真正的工程量与风险不在翻译，而在**字体字形**：现有 `glyph ranges` 只到 `U+00FF`，且所用字体不含汉字，不解决会整屏方块。
5. ImGui 版本为 **1.91.9**，属于动态字体纹理（1.92）之前，所有字形必须启动时一次性光栅化 → 字形集必须做精简，不能全量。

---

## 2. i18n 架构

### 2.1 文件位置

| 文件 | 体积 | 作用 |
|---|---|---|
| `runtime/ui/i18n.h` | 978 B | 仅声明两个接口：`const char *T(const char *en)`、`void ui_upper(std::string&)` |
| **`runtime/ui/i18n.mm`** | 3.9 KB | **核心**：两张表、`lookup()`、`T()`、`settings_ui_language()` 系统语言探测 |
| `runtime/ui/i18n_pt_br.h` | 75 KB | 葡语表，**785 条**，格式 `{"English", "Português"}` |
| `runtime/ui/i18n_multi.h` | 19 KB | 西/法/德/意/荷表，**94 条**，格式 `{en, es, fr, de, it, nl}` |

### 2.2 调用链

```
界面代码 ImGui::Button(T("Back"))
        │
        ▼
T(en)                    (i18n.mm)
  ├─ settings_ui_language()   ← 读 g_settings.ui_language；AUTO 时用 SDL_GetPreferredLocales 探测
  ├─ lang == EN  → 直接返回英文原文
  ├─ lang == PT_BR → 查 kPtBr（unordered_map<string_view, const char*>）
  └─ 其他 → 查 kMulti，取 tr[lang - UI_LANG_ES]
        │
        ▼
返回译文指针（未命中则返回英文原文）
```

### 2.3 两条表的结构差异（关键约束）

```cpp
// i18n.mm : 20
struct Pair  { const char *en, *pt;                    };  // 二元组，葡语专用
struct Multi { const char *en, *tr[5];                 };  // 六元组 = en + es/fr/de/it/nl，**固定 5 列**
```

```cpp
// i18n.mm : 36-41  查找逻辑
if (lang == UI_LANG_PT_BR) { ... map->find(en) ... }
auto it = multi->find(en);
return it == multi->end() || !*it->second->tr[lang - UI_LANG_ES]
       ? NULL : it->second->tr[lang - UI_LANG_ES];
```

> **推论**：`kMulti` 是固定 5 列，且索引方式是 `tr[lang - UI_LANG_ES]`。若把中文塞进 `kMulti`，既要扩列（94 行 × 6 字段全补），又要改索引公式，且新枚举值会越界。→ **中文必须单独建表**。

### 2.4 `##id` 约定

```cpp
ImGui::Button(T("Play##start"));   // "##" 前可见文本参与翻译，"##" 后的 ID 原样保留
```

`T()` 用 `strstr(en, "##")` 切分：翻译可见部分，再把 `##id` 拼回（结果存入 per-language 缓存保证指针生命期）。

---

## 3. 改造清单（按依赖顺序，共 6 处）

### ① 语言枚举 — `runtime/settings.h:34`

```cpp
// 现状
enum { UI_LANG_AUTO = 0, UI_LANG_EN = 1, UI_LANG_PT_BR = 2, UI_LANG_ES = 3, UI_LANG_FR = 4,
       UI_LANG_DE = 5, UI_LANG_IT = 6, UI_LANG_NL = 7, N_UI_LANGUAGES = 8 };

// 目标
enum { ..., UI_LANG_NL = 7, UI_LANG_ZH_HANS = 8, UI_LANG_ZH_HANT = 9, N_UI_LANGUAGES = 10 };
```

配套：`runtime/settings.c:254` 的边界校验是 `atoi(val) < N_UI_LANGUAGES`，随枚举自动生效，**无需修改**。

### ② 界面语言下拉框 — `runtime/ui/difficulty.mm:72`

```cpp
static const char *const ui_names[N_UI_LANGUAGES] =
    {"Auto", "English", "Português (Brasil)", "Español", "Français", ...};
if (opt_choice("App language", &g_settings.ui_language, ui_names, N_UI_LANGUAGES, UI_LANG_AUTO, ...))
```

- 数组下标**必须与枚举值严格一一对应**，尾部追加 `"简体中文"`、`"繁體中文"`。
- 语言名使用母语自称（endonym），**不要再套 `T()`**，否则切到中文后其他语言名也会被翻成中文。

### ③ 新建中文表 — `runtime/ui/i18n_zh_hans.h` / `i18n_zh_hant.h`

- 照 `i18n_pt_br.h` 的二元组格式：`{"English", "中文"}`，**不要动 `i18n_multi.h`**。
- 语料基线：以 `i18n_pt_br.h` 的 785 条英文 key 为准（它是目前覆盖最全的语言）。
- 必须包含语言名自身的条目（参考 `i18n_pt_br.h:217` 的 `{"English", "Inglês"}`），否则下拉框内语言名不随界面语言变化。

### ④ 改 `runtime/ui/i18n.mm`（三处小改）

```cpp
// (a) 引入新表
static const Pair kZhHans[] = { #include "i18n_zh_hans.h" };
static const Pair kZhHant[] = { #include "i18n_zh_hant.h" };

// (b) lookup() 增加分支（照 PT_BR 写法，独立 map）
if (lang == UI_LANG_ZH_HANS) { /* zh_map->find(en) */ }
if (lang == UI_LANG_ZH_HANT) { /* hant_map->find(en) */ }

// (c) 系统语言探测（i18n.mm:84）—— 见下方说明
```

**(c) 是重点**。现有探测：

```cpp
static const struct { const char *code; int lang; } codes[] =
    {{"pt", UI_LANG_PT_BR}, {"es", UI_LANG_ES}, ...};
if (!strcmp(l[0]->language, codes[i].code)) sys = codes[i].lang;
```

只比对 `SDL_Locale::language`（两字母码）。**中文简繁的 language 都是 `"zh"`**，必须额外读 `l[0]->country`：

| country | 目标 |
|---|---|
| `CN` / `SG` | 简体 |
| `TW` / `HK` / `MO` | 繁体 |
| 空 / 其他 | 简体（默认） |

不改这里，Auto 在中文系统上会静默落到英文。

### ⑤ 字体与字形（详见第 4 节）

- 启动器：`runtime/ui/launcher.mm:1687–1690`
- 设置菜单：`runtime/ui/menu.mm:379–404`

### ⑥ 明确不需要改

`runtime/settings.c:69` 的 `g_language_names[]` 是**游戏语言**名单（已含 `"Chinese (Simplified)"` / `"Chinese (Traditional)"`），与界面语言是两套独立枚举（`LANG_*` vs `UI_LANG_*`），不要混用。

---

## 4. 字体障碍（本次调研的核心风险）

### 4.1 现状

```cpp
// runtime/ui/launcher.mm:1687
static const ImWchar ranges[] = {0x20, 0xFF, 0x2039, 0x203A, 0x2192, 0x2192, 0x2713, 0x2713, 0x25B8, 0x25B8, 0};
```

- 字形范围仅 **U+0020–U+00FF** + `‹ › → ✓ ▸` 五个符号。
- 字体文件为 `Baloo2-SemiBold.ttf`（OFL，随包内置）与 `DejaVuSans.ttf`——**均不含汉字字形**。
- `menu.mm` 的 `ImFontConfig cfg` 未显式设置 `GlyphRanges`，走 ImGui 默认（同样仅拉丁区）。

> 结论：只替换字符串而不动字体，界面将整屏显示方块（tofu）。

### 4.2 约束：ImGui 1.91.9 无动态字体纹理

| 项 | 值 |
|---|---|
| `IMGUI_VERSION` | `1.91.9` |
| 动态字体纹理（`ImGuiBackendFlags_RendererHasTextures`） | 1.92 才引入，**当前不可用** |

也就是说字形必须在 `AddFontFromFileTTF` 时一次性光栅化进 atlas，运行时无法按需补字。因此：

- **禁止使用 `GetGlyphRangesChineseFull()`**（约 2 万字）；按 3 档字号 × 2 套 UI 计算，atlas 会膨胀到不可接受。
- **推荐 `ImFontGlyphRangesBuilder` 动态构建**：把中文翻译表中实际出现的全部字符去重后构入 range。启动器文案规模有限，实际用字预计 **300–600 个**，atlas 压力可降两到三个数量级。

### 4.3 实现要点

1. **字体来源**：必须使用 OFL 类可再分发字体（思源黑体 / Noto Sans SC / Noto Sans TC），随 APK 打包，取用方式仿照现有 `rt_resource("Baloo2-SemiBold.ttf", "third_party/fonts/Baloo2/...")`。
2. **不得**使用 Windows/macOS 系统字体或任何带版权的商业中文字体。
3. **字号实例**（每个都要一份中文 glyph，需统计总数）：

| 位置 | 变量 | 字号 |
|---|---|---|
| launcher | `g_font_small` | 30（Baloo2 时 ×1.75） |
| launcher | `g_font_menu` | 52（×1.85） |
| launcher | `g_font_title` | 92（×2.3） |
| menu | `g_font` / `g_font_big` | 34 / 56 |
| menu | `g_menu_font` / `g_menu_font_big` | 39×kUi / 72×kUi |

4. **标点与符号**：中文文案中的 `「」`、`：`、`，` 等全角符号需在 range 内，且所选字体需覆盖。

---

## 5. 里程碑建议

| 阶段 | 内容 | 验收标准 |
|---|---|---|
| **M1** 枚举接线 | 改 `settings.h` + `difficulty.mm`，下拉框出现“简体中文”，选中后界面仍英文 | 语言可选中、可持久化（`ui_language` 写入配置并重启后保持） |
| **M2** 字体打通 | 接入 CJK 字体 + `ImFontGlyphRangesBuilder`；仅翻译 3–5 条（`Play` / `Settings` / `Quit`） | 显示正常无方块、图集尺寸与首帧加载时间在可接受范围 |
| **M3** 简体铺量 | 按 785 条英文 key 逐条翻译，优先菜单高频项 | 遍历所有设置页，无串行、无溢出、无 `##id` 冲突 |
| **M4** 繁体铺量 | 可用 OpenCC 由简转繁打底，术语人工复核 | 简繁切换均正常 |
| **M5** 上游提交 | 先开 Issue 询问是否接收中文 PR | 遵循作者模板，PR 只含代码与语言表 |

> 时机提示：1.2.0 刚新增 5 种语言，作者当前处于愿意接收语言 PR 的窗口期。

---

## 6. 细节与坑位清单

1. **`##id` 必须保留**：`"Play##start"` → `"开始游戏##start"`。丢失 `##` 后半段会导致 ImGui 控件 ID 冲突，出现点击 A 触发 B。
2. **`ui_upper()` 无需改造**：只处理 Latin-1（`0xC3 0xA0..0xBE`）大写映射，中文原样返回。
3. **文本更宽**：中文同义文案宽度约为英文 1.6–1.8 倍，菜单行宽固定，需要 `CalcTextSize` 截断或精简措辞。
4. **保持 key 为英文原文**：`option_rows.mm` 的 label/value/desc 均以英文 label 作为 ID（`T()` 只翻显示层），改动 key 会断掉查表。
5. **APK 签名变更**：1.2.0 更换了发布密钥，自建测试包无法覆盖安装官方版，测试需先导出存档或改用独立 applicationId。
6. **空槽位判断**：`lookup()` 中有 `!*it->second->tr[...]` 的空串判断，独立建表后不受影响，但仍需保证表中无空译文。

---

## 7. 复现本次调研

```bash
git clone --depth 1 --filter=blob:none --no-checkout \
  https://github.com/Nycolazs/LASRecomp-Android-Plus.git las
cd las

# 定位 i18n 相关文件
git ls-tree -r --name-only HEAD | grep -Ei "ui/|lang|i18n|translat"

# 读取核心实现
git show HEAD:runtime/ui/i18n.mm
git show HEAD:runtime/ui/i18n.h

# 查语言枚举与下拉框
git show HEAD:runtime/settings.h | grep -n -A3 "UI_LANG_AUTO"
git show HEAD:runtime/ui/difficulty.mm | sed -n '70,80p'

# 查字形范围（关键）
git show HEAD:runtime/ui/launcher.mm | sed -n '1678,1692p'
git show HEAD:runtime/ui/menu.mm     | sed -n '376,406p'
```

---

## 8. 待确认 / 未覆盖项

| 项 | 说明 |
|---|---|
| 中文译本语料 | 尚无现成中文翻译，785 条为待翻译工作量（本报告未产出译文） |
| 实际用字数统计 | 需在译文产出后统计去重字符数，才能定 glyph range 与 atlas 预算 |
| Android 侧 locale 注入 | 未验证 `LasActivity` 是否向 SDL 传递系统 locale（若未传递，Auto 探测需额外接 JNI 取 `Locale.getDefault()`） |
| 图集内存实测 | 需在目标机型（如骁龙 8 系）实测首帧加载耗时与显存占用 |
| 与 PT-BR 补丁机制的关系 | `translations/pt-BR` 属游戏内文本补丁，本次不涉及，勿混入同一 PR |

---

*本报告为阶段性调研成果，结论基于 commit `cca791a` 的静态代码阅读；行号在后续版本中可能偏移，实施前请以实际代码复核。*
