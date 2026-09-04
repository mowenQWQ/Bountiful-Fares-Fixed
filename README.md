# Bountiful Fares Fixed

**Bountiful Fares 1.3.0-1.20.1 修复版（手动构建）——修复 1.20.1 狼乞食 NPE 服务器崩溃。**
**A hand-built fixed build of Bountiful Fares 1.3.0-1.20.1 — fixes the wolf begging NPE server crash on 1.20.1.**

[中文](#中文) | [English](#english)

---

## 中文

### Bountiful Fares 1.3.0-1.20.1（手动构建修复版）

> 一个修复了 1.20.1 版本狼乞食崩溃（WolfEntityMixin NPE）的 Bountiful Fares 构建。
> 基于官方仓库 2025-02-12 干净的 1.20.1 源码 + 从 1.3.0 分支复刻的 wolf 喂食修复。

- **Mod**: [Bountiful Fares](https://modrinth.com/mod/bountiful-fares)（Fabric）
- **游戏版本**: Minecraft 1.20.1
- **模组加载器**: Fabric（可通过 Sinytra Connector 在 Forge 服务端运行）
- **版本**: `1.3.0-1.20.1`（修复构建）
- **许可证**: [MIT-0](#中文)

---

### 🐛 修复了什么

#### 崩溃：狼乞食导致服务器崩溃（NPE）

**现象**：玩家手持一个**非食物物品**站在狼旁边时，狼的 `BegGoal`（乞食 AI）触发，服务器主线程抛出 `NullPointerException`：

```
java.lang.NullPointerException: Cannot invoke "FoodProperties.isMeat()" because
the return value of "Item.getFoodProperties()" is null
        at Wolf.handler$zla000$bountifulfares$feedMulchToWolf(Wolf.java:577)
        （wolf BegGoal → RandomStrollGoal → 实体 tick）
Description: Ticking entity
```

**根因**：原版 1.2.1 通过 `WolfEntityMixin.feedMulchToWolf` 把"覆盖物（mulch）"直接注入原版狼的 `isBreedingItem` 判断。该 mixin 无条件调用 `item.getFoodComponent().isMeat()`——当玩家手里拿的是**不是食物**的物品（如剑、泥土）时，`getFoodComponent()` 返回 `null` → 对 null 调 `isMeat()` → NPE → 一整只狼的实体 tick 崩溃 → 服务端整个崩掉。

**修复方式**（与上游 1.3.0 做法一致，数据驱动而非代码注入）：
1. **删除** `WolfEntityMixin`（mixin 提交里移除 + 源文件删除）
2. **新增** `data/minecraft/tags/item/wolf_food.json`：将 `#bountifulfares:mulch` 挂到原版 `minecraft:wolf_food` 物品标签下
3. 原版狼的喂食/乞食逻辑本身就识别 `#minecraft:wolf_food` 标签且自带空值判断——覆盖物依然能喂狼，但不再走自定义 mixin，天然免疫 NPE

**效果**：
- ✅ 狼吃覆盖物功能保留（`walnut_mulch`/`walnut_mulch_block`/`palm_mulch`/`palm_mulch_block` 仍可喂狼）
- ✅ 玩家手持任意物品时狼乞食不再崩溃
- ✅ 不再有任何对 `getFoodComponent()` 的无保护调用

---

### 🧱 为什么需要手动构建

上游在该分支上的版本情况：

| 版本 | 状态 |
|---|---|
| `1.2.1-1.20.1` | 官方发布的 1.20.1 最高版，**含此崩溃 bug** |
| `1.3.0-1.20.1` | 源码中已修复，**但官方从未发布 1.20.1 的 jar** |
| `1.20.1` 分支（2025-03 后） | 作者把 1.21 代码搬回 1.20.1（`move 1.21 to 1.20`），**分支被 1.21 API 污染、无法编译** |

因此本构建：
- 取官方 `1.20.1` 分支在 **2025-02-12**（commit `acf5250b`，"Fix compat when using synitra"）前的干净源码——该版本可正常编译，且是 `1.3.0-1.20.1` 的开发基线
- 从最新版源码中复刻 wolf 修复（wolf_food tag + 移除 mixin）
- 用 **JDK 21 + Gradle 8.6 + fabric-loom 1.6.12** 官方标准流程构建

---

### 📦 安装

直接 `git clone` 本仓库取 `bountifulfares-1.3.0-1.20.1.jar`（或到 Release 页下载），放到服务端/客户端的 `mods/` 目录，替换旧的 `bountifulfares-1.2.1-1.20.1.jar`（或先备份旧文件）：

```
bountifulfares-1.3.0-1.20.1.jar  →  mods/
```

- 本构建是 **Fabric** 版，纯 fabric 服务端可直接用；Forge 服务端需通过 **Sinytra Connector** 运行（Connector 会把 Fabric mod 转译到 Forge）。
- 依赖：`fabric-api`、`terraform-wood-api-v1`（已内嵌 include 到 jar）。

---

### 🔨 自行构建

```bash
git clone --single-branch --branch 1.20.1 https://github.com/Heccology/Bountiful-Fares.git
cd Bountiful-Fares
git checkout acf5250b80276093e0cb76cd527aea4a860460d6   # 干净的 1.3.0 开发基线

# 复刻 wolf 修复：
# 1) 删除 src/main/java/net/hecco/bountifulfares/mixin/WolfEntityMixin.java
# 2) 从 src/main/resources/bountifulfares.mixins.json 移除 "WolfEntityMixin" 条目
# 3) 新增 src/main/resources/data/minecraft/tags/item/wolf_food.json:
#    { "values": [ "#bountifulfares:mulch" ] }

./gradlew build -x test   # JDK 21 + Gradle 8.6（国内网络可换腾讯云/阿里云镜像）
# 产物: build/libs/bountifulfares-1.3.0-1.20.1.jar
```

---

### 📜 许可证

本构建（Bountiful Fares 修复版）以 **MIT-0**（MIT No Attribution）协议发布——详见 [LICENSE](LICENSE)。

⚠️ 上游 [Bountiful Fares](https://github.com/Heccology/Bountiful-Fares) 自身的许可证以官方仓库为准（其仓库渲染显示为 MIT；分发请自行核对上游 LICENSE）。

**MIT No Attribution（MIT-0）**：任何人可自由使用、复制、修改、合并、发布、再分发、销售本软件副本，无需署名，无需保留版权声明；软件按"现状"提供，不附带任何明示或默示担保。

---

### 🧪 验证记录

- 构建：`BUILD SUCCESSFUL`（fabric-loom 1.6.12 / Gradle 8.6 / JDK 21）
- jar 内 `mixins.json` 合法，无 `WolfEntityMixin`
- jar 内含 `data/minecraft/tags/item/wolf_food.json`
- 服务器实测：狼乞食不再崩溃（2026-09-04）

---

## English

### Bountiful Fares 1.3.0-1.20.1 (Hand-built Fixed Build)

> A rebuild of Bountiful Fares that fixes the wolf begging crash (WolfEntityMixin NPE) on 1.20.1.
> Based on the clean 1.20.1 sources from the official repo (as of 2025-02-12) + the wolf-feeding fix back-ported from the 1.3.0 branch.

- **Mod**: [Bountiful Fares](https://modrinth.com/mod/bountiful-fares) (Fabric)
- **Game version**: Minecraft 1.20.1
- **Mod loader**: Fabric (works on Forge servers via Sinytra Connector)
- **Version**: `1.3.0-1.20.1` (fixed build)
- **License**: [MIT-0](#english)

---

### 🐛 What was fixed

#### Crash: wolf begging causes a server crash (NPE)

**Symptom**: When a player stands near a wolf holding a **non-food item**, the wolf's `BegGoal` (begging AI) triggers and the server main thread throws a `NullPointerException`:

```
java.lang.NullPointerException: Cannot invoke "FoodProperties.isMeat()" because
the return value of "Item.getFoodProperties()" is null
        at Wolf.handler$zla000$bountifulfares$feedMulchToWolf(Wolf.java:577)
        (wolf BegGoal → RandomStrollGoal → entity tick)
Description: Ticking entity
```

**Root cause**: Upstream 1.2.1 injects "mulch" feeding into the vanilla wolf's `isBreedingItem` check via `WolfEntityMixin.feedMulchToWolf`. That mixin unconditionally calls `item.getFoodComponent().isMeat()` — when the held item is **not food** (e.g. a sword, dirt), `getFoodComponent()` returns `null` → calling `isMeat()` on null → NPE → the wolf's entity tick crashes → the whole server goes down.

**Fix** (same approach as upstream 1.3.0: data-driven instead of code injection):
1. **Removed** `WolfEntityMixin` (removed from the mixins config + the source file deleted)
2. **Added** `data/minecraft/tags/item/wolf_food.json`: attach `#bountifulfares:mulch` to the vanilla `minecraft:wolf_food` item tag
3. The vanilla wolf's feeding/begging logic already recognizes the `#minecraft:wolf_food` tag with built-in null handling — wolves can still be fed mulch, but without the custom mixin, so the NPE is impossible by construction

**Result**:
- ✅ Wolf-eats-mulch behavior preserved (`walnut_mulch`/`walnut_mulch_block`/`palm_mulch`/`palm_mulch_block` still feed wolves)
- ✅ Wolves begging no longer crash the server no matter what item the player holds
- ✅ No unprotected `getFoodComponent()` calls remain

---

### 🧱 Why a hand-built rebuild is needed

The state of the upstream 1.20.1 branch:

| Version | Status |
|---|---|
| `1.2.1-1.20.1` | The latest official 1.20.1 release, **contains this crash bug** |
| `1.3.0-1.20.1` | Fixed in source, **but the author never published a 1.20.1 jar** |
| `1.20.1` branch (after 2025-03) | The author moved 1.21 code back into 1.20.1 (`move 1.21 to 1.20`), **the branch is polluted by 1.21 APIs and won't compile** |

So this build:
- Takes the clean sources of the official `1.20.1` branch as of **2025-02-12** (commit `acf5250b`, "Fix compat when using synitra") — this version compiles and is the dev baseline of `1.3.0-1.20.1`
- Back-ports the wolf fix from the latest sources (wolf_food tag + mixin removal)
- Builds with the standard toolchain: **JDK 21 + Gradle 8.6 + fabric-loom 1.6.12**

---

### 📦 Installation

`git clone` this repo and grab `bountifulfares-1.3.0-1.20.1.jar` (or download from the Release page), then put it into the `mods/` directory of your server/client, replacing the old `bountifulfares-1.2.1-1.20.1.jar` (or back it up first):

```
bountifulfares-1.3.0-1.20.1.jar  →  mods/
```

- This is a **Fabric** build: works directly on vanilla Fabric servers; Forge servers need **Sinytra Connector** (which translates Fabric mods onto Forge).
- Dependencies: `fabric-api` and `terraform-wood-api-v1` (already jar-in-jar'd inside).

---

### 🔨 Build it yourself

```bash
git clone --single-branch --branch 1.20.1 https://github.com/Heccology/Bountiful-Fares.git
cd Bountiful-Fares
git checkout acf5250b80276093e0cb76cd527aea4a860460d6   # clean 1.3.0 dev baseline

# Reproduce the wolf fix:
# 1) Delete src/main/java/net/hecco/bountifulfares/mixin/WolfEntityMixin.java
# 2) Remove the "WolfEntityMixin" entry from src/main/resources/bountifulfares.mixins.json
# 3) Add src/main/resources/data/minecraft/tags/item/wolf_food.json:
#    { "values": [ "#bountifulfares:mulch" ] }

./gradlew build -x test   # JDK 21 + Gradle 8.6
# Output: build/libs/bountifulfares-1.3.0-1.20.1.jar
```

---

### 📜 License

This build (Bountiful Fares Fixed) is released under **MIT-0** (MIT No Attribution) — see [LICENSE](LICENSE).

⚠️ The upstream [Bountiful Fares](https://github.com/Heccology/Bountiful-Fares) project's own license is defined in its official repository (GitHub renders it as MIT; verify the upstream LICENSE yourself before redistributing).

**MIT No Attribution (MIT-0)**: anyone is free to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of this software, with no attribution or copyright notice required; the software is provided "as is", without warranty of any kind.

---

### 🧪 Verification

- Build: `BUILD SUCCESSFUL` (fabric-loom 1.6.12 / Gradle 8.6 / JDK 21)
- The jar's `mixins.json` is valid and contains no `WolfEntityMixin`
- The jar contains `data/minecraft/tags/item/wolf_food.json`
- Live-tested on a server: wolf begging no longer crashes (2026-09-04)

---

*Built: 2026-09-04 · Based on upstream [Heccology/Bountiful-Fares](https://github.com/Heccology/Bountiful-Fares) sources*

---

## 🤖 AI 使用声明 / AI Usage Disclosure

本项目在开发与维护过程中使用了 AI 编程助手（Claude / Anthropic）辅助代码编写、文档整理与问题排查；核心决策、内容审核与最终发布由维护者完成。

This project was developed and maintained with the assistance of an AI coding assistant (Claude / Anthropic) for coding, documentation, and troubleshooting. Core decisions, content review, and final releases are made by the maintainer.
