---
name: revit-material-access-creation
description: |
  需要以编程方式检索、创建或复制 Revit 材料（Material），或为材料挂接结构/热工"属性资源"（PropertySetElement/ThermalAsset）时调用。Trigger：create/copy/duplicate material、材质库、复制材料、新建材质、热工属性、ThermalAsset、PropertySetElement。不适用于：读取某个图元实际使用的材料（改用材料检索回退 skill）、修改填充图案本身。
source_book: 《API开发指南 Autodesk Revit》 宦国胜
source_chapter: 第3章 3.9（约p242–p245）
tags: [revit-api, material, propertysetelement, thermal-asset, element]
related_skills:
  - slug: revit-material-retrieval-fallback
    relation: contrasts-with
  - slug: revit-compound-structure-layers
    relation: composes-with
---

# 材料（Material）对象访问与创建流程（含 PropertySetElement 属性资源）

## R — 原文 (Reading)

> 在 Revit 平台 API 中，会存储材料数据并作为 Element 来管理。…… API 中有两种方法可创建新的 Material 对象：复制现有 Material；添加新 Material。
>
> — 宦国胜, 第3章 3.9 材料（约p242–p245）

---

## I — 方法论骨架 (Interpretation)

材料在 Revit API 中不是独立的轻量对象，而是一个标准的 Element——因此一切 Element 检索手段对它都有效，且它的属性被拆成两层：

1. **直接属性层**：颜色、填充图案、渲染外观等"图形属性"直接挂在 `Material` 类的属性上，读写即得。
2. **属性资源层**：结构、热工这类"工程属性"不存放在 Material 自身，而是存放在独立的 PropertySetElement（属性资源）中，Material 通过 `SetMaterialAspectByPropertySet(MaterialAspect, pseId)` 按方面（Structure/Thermal 等）间接关联。

创建材料有两条路：
- `Material.Duplicate(newName)`——复制现有材料，新对象与原对象同类型，适合"在既有材料基础上改"。
- `Material.Create(document, name)`——从零新建，适合需要全新材料且不依赖任何样板的情况。

想给材料挂热工数据，就要先创建/找到承载 ThermalAsset 的 PropertySetElement，再通过 SetMaterialAspectByPropertySet 关联——"Duplicate 保类型 + PropertySetElement 承载属性"是这个体系的关键结构。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 批量创建带热工属性的副本材料（节能检查）
- **问题**: 节能检查需要一批与现有材料对应、但热工参数不同的副本材料。
- **方法论的使用**: 用 `Material.Duplicate(newName)` 复制（副本与原对象同类型），再通过 PropertySetElement + `SetMaterialAspectByPropertySet(MaterialAspect.Thermal, pse.Id)` 关联 ThermalAsset，把热工属性挂到副本上。
- **结论**: Duplicate 通道天然保住类型一致性，属性资源通道把热工数据与图形数据解耦，两条通道各司其职。
- **结果**: 副本材料既保留原图形外观，又携带新热工数据，可参与节能计算。

### 案例 2: 材料的三种检索方式
- **问题**: 在程序中需要定位某个 Material 对象。
- **方法论的使用**: 利用 Material 是 Element 的事实，走标准检索：`FilteredElementCollector.OfClass(typeof(Material))` 全量过滤；或按 `Name` 匹配；或经 `document.Settings.Materials` 集合访问。
- **结论**: 三种入口覆盖"已知类型/已知名称/要整个材料库"三种典型需求。
- **结果**: 任何材料定位场景都能落到其中一条入口，不需要专门的查找 API。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 写插件时需要新建或复制材料（如为不同构造做法批量生成材料变体）。
2. 要为材料附加/替换热工或结构"属性资源"（ThermalAsset、结构属性），做节能或结构数据准备。
3. 需要列出项目材料库、按名称查找材料并修改颜色/填充图案。
4. 用户问"材质和它的热属性在 API 里是什么关系、怎么挂上"。

### 语言信号 (用户的话里出现这些就应激活)

- "创建/复制一个材质" / "create / duplicate a material"
- "给材料加热工属性" / "set ThermalAsset / thermal properties on a material"
- "PropertySetElement / SetMaterialAspectByPropertySet 怎么用"
- "材质库怎么遍历" / "iterate document materials / Settings.Materials"

### 与相邻 skill 的区分

- 与 `revit-material-retrieval-fallback` 的区别：本 skill 处理材料对象本身的创建/复制与属性资源挂接；该 skill 处理“某个图元实际用的是哪个材料”的反查回退，方向相反。
- 与 `revit-compound-structure-layers` 的关系：复合结构层只通过 MaterialId 引用材料，是材料的消费端；本 skill 是被引用材料的创建端，两者互补。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后, agent 应按以下步骤执行:

1. **定位材料**
   - 按需选择入口：`OfClass(typeof(Material))` / 按 Name 匹配 / `doc.Settings.Materials`。
   - 完成标准: 已拿到目标 `Material` 对象，或确认材料库中不存在该名称。
   - 判停条件: 若目标材料不存在且用户不需要新建，报告后结束。

2. **创建材料**
   - 有可参照的既有材料 → `sourceMaterial.Duplicate(newName)`；需全新材料 → `Material.Create(doc, name)`。
   - 完成标准: 得到新 Material 的引用且其 ElementId 有效。

3. **挂接属性资源（按需）**
   - 为 Thermal/Structure 方面创建或定位 PropertySetElement，再 `material.SetMaterialAspectByPropertySet(MaterialAspect.Thermal, pse.Id)`。
   - 完成标准: 从材料侧读回对应 aspect 的 PropertySetElement 与预期一致。
   - 判停条件: 若只需改颜色/填充图案等图形属性，直接写 `material.Color` 等属性，跳过本步。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 想知道"某堵墙/某个楼板用的什么材料"——这是图元→材料的检索问题，走层级回退策略 skill，不是本 skill 的创建/检索材料对象问题。
- 修改填充图案对象本身（FillPatternElement）——那是另一个对象体系。
- 材料外观资产（AppearanceAsset）的渲染细节，本书基于 2014 API 覆盖有限。

### 作者在书中警告的失败模式

- 以为热工/结构属性像颜色一样直接挂在 Material 上——实际必须经 PropertySetElement 间接关联，直接找属性会找不到。
- 创建带属性资源的新材料时忘记 Duplicate 后属性资源不会自动复制到位，需显式重建关联。

### 作者的盲点 / 时代局限

- 本书基于 Revit 2014 / .NET 4.0。新版中 Material 的 API（如外观资产、PhysicalAsset 体系）有扩充，套用旧接口前需核对版本。
- 2014 时代对材料库跨文档共享（MaterialLibrary）覆盖较少。

### 容易混淆的邻近方法论

- 图元材料检索回退策略（本批次另一 skill）——方向相反：那是"从图元找材料"。
- 复合结构层材料（CompoundStructureLayer.MaterialId）——是材料的一个消费场景，不是材料创建。

---

## 相关 skills

- **revit-material-retrieval-fallback**（图元材料检索的层级回退策略（含空值与不可访问陷阱） · contrasts-with）— 该 skill 从图元反查实际使用的材料，本 skill 负责材料对象的创建/复制与属性资源挂接，方向相反。
- **revit-compound-structure-layers**（复合结构 CompoundStructure 层读取与修改流程 · composes-with）— 复合结构层的 MaterialId 引用本 skill 创建的材料对象，本 skill 是被引用对象的创建端，配合构成“创建→消费”链路。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 留待阶段4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-08-27
