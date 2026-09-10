# Revit 二次开发术语词典（GLOSSARY）

> 来源：《API开发指南 Autodesk Revit》（宦国胜，2016，基于 Revit 2014 / .NET 4.0）蒸馏出的 159 个原子 skill（`books/revit-api/<slug>/SKILL.md`）。
> 用途：所有 skill 与读者共用的共享术语表；定义忠于原书语义，用编者自己的话表述（20-60 字）。
> 排序：主列表按英文术语首字母排序；每条挂 1-4 个真实存在的 skill slug（经 `_stage2/skill-map.yaml` 核对）。
> 带 ⚠️ 版本提示 的词条：该术语带 Revit 2014 时代背景，新版中已发生结构性变化。

---

## A

### AddInCommandBinding（外接程序命令绑定）
- 中文译名：命令绑定
- 一句话定义：把 Revit 内置命令绑定到自定义处理逻辑，从而拦截、重写或禁用该命令的默认行为。
- 相关 skill：`revit-addincommandbinding-override-commands`、`revit-postcommand-single-limit`

### AnalyticalLink（分析链接）
- 中文译名：分析链接
- 一句话定义：连接两个分析图元的抽象连接对象，端点由 Hub 锚点定位，创建时传 Hub 的 ElementId。
- 相关 skill：`revit-analytical-link-hub`、`revit-analytical-model-geometry`

### AnalyticalModel（分析模型）
- 中文译名：分析模型
- 一句话定义：结构图元供分析计算用的抽象几何，按类型分流读取：基础 GetPoint、柱梁 GetCurve、墙 GetCurves；调用前须先 IsSinglePoint/IsSingleCurve 探测，否则抛 InapplicableDataException。
- 相关 skill：`revit-analytical-model-geometry`、`revit-analytical-model-supports`、`revit-analytical-model-guard-clause`、`revit-analytical-model-exception`
- ⚠️ 版本提示：三件套为 Revit 2014 形态，新版已重构为独立分析图元体系。

### Application / UIApplication（应用程序对象）
- 中文译名：应用程序
- 一句话定义：Revit 会话级对象两两配对（应用/文档 × DB/UI 层）：DB 层管数据，UI 层管界面能力。
- 相关 skill：`revit-document-vs-uidocument`、`revit-db-application-scenarios`、`revit-plugin-entry-types`

### Arc（圆弧）
- 中文译名：圆弧
- 一句话定义：参数化圆弧曲线；轴网等曲线经 IsCurved 判定后由 Curve 转型读取半径等属性。
- 相关 skill：`revit-grid-curve-creation`、`revit-geometry-utility-classes`

### Assembly / Part / PartMaker（部件与零件）
- 中文译名：部件/零件/零件分割器
- 一句话定义：构造建模机制：Assembly 把多图元组成装配体并生成部件视图，PartMaker 按规则把图元分割为 Part。
- 相关 skill：`revit-assembly-part-modeling`、`revit-partmaker-term`

---

## B

### BeamSystem（梁系统）
- 中文译名：梁系统
- 一句话定义：聚合结构图元，自身不挂 AnalyticalModel，分析信息须从成员图元获取。
- 相关 skill：`revit-beamsystem-analytical-absence`、`revit-analytical-model-geometry`

### Binding（绑定）
- 中文译名：参数绑定
- 一句话定义：共享参数与图元类别的关联方式：TypeBinding 全类型共享值，InstanceBinding 每实例独立。
- 相关 skill：`revit-binding-type-vs-instance`、`revit-binding-insert-silent-fail`

### BuiltInParameter（内建参数）
- 中文译名：内建参数
- 一句话定义：Revit 预定义的参数枚举，跨语言稳定；部分只读属性需绕道参数写入实现修改。
- 相关 skill：`revit-builtin-parameter-semantics`、`revit-parameter-index-lookup`

### BoundingBox 过滤器族（BoundingBoxIsInside / Intersects / ContainsPoint）
- 中文译名：边界框过滤器
- 一句话定义：按图元包围盒与 Outline（最小/最大点框）关系做空间初筛的快速过滤器，异形构件会误报。
- 相关 skill：`revit-boundingbox-filter-tradeoffs`、`revit-filter-quick-slow-logical`

---

## C

### Cancelable Events（可取消事件）
- 中文译名：可取消事件
- 一句话定义：DocumentClosing/DocumentSaving 等操作前发出的事件族，处理函数可取消后续操作；事件期间不可编辑模型。
- 相关 skill：`revit-cancelable-events-propagation`、`revit-readonly-event-checks`、`revit-documentclosing-no-model-edit`

### Category（类别）
- 中文译名：类别
- 一句话定义：类型-实例四层模型的顶层归类（墙/门/窗），全集从 Document.Settings.Categories 读取。
- 相关 skill：`revit-category-family-symbol-instance`、`revit-element-six-groups`

### CheckoutElements / WorksharingUtils（检出）
- 中文译名：检出/放弃所有权
- 一句话定义：工作共享下批量获取或让出图元/工作集所有权的工具方法，必须大批量一次性调用。
- 相关 skill：`revit-worksharing-api-decision-framework`、`revit-checkout-group-propagation`

### CompoundStructure（复合结构）
- 中文译名：复合结构
- 一句话定义：挂在墙/楼板/屋顶类型上的多层构造定义（材料/厚度/功能），类型级修改影响全部实例。
- 相关 skill：`revit-compound-structure-layers`、`revit-wall-location-line`

### Connector / ConnectorManager（连接件）
- 中文译名：MEP 连接件
- 一句话定义：MEP 设备与管线的接口对象，携带域/流向/系统归属，经 AllRefs 可递归遍历物理拓扑。
- 相关 skill：`revit-mep-connector-system-topology`、`revit-mep-diameter-param`、`revit-mep-curve-creation-entries`

### CopyElements（复制图元）
- 中文译名：复制图元
- 一句话定义：ElementTransformUtils 的批量复制入口，与单图元复制语义不同，注意组复制的联动。
- 相关 skill：`revit-element-copy-decision`、`revit-element-transform-utils`

### CurveByPoints（由点生成的曲线）
- 中文译名：点驱动的曲线
- 一句话定义：概念环境中把一组参照点连成参数化曲线，移动点则曲线跟随更新。
- 相关 skill：`revit-referencepoint-curvebypoints`、`revit-conceptual-forms-type-selection`

### Curve（曲线）
- 中文译名：曲线
- 一句话定义：几何 API 的参数化曲线基类，Line/Arc 经 IsCurved 判定转型；Edge 亦可展开为 Curve。
- 相关 skill：`revit-face-edge-loop-traversal`、`revit-geometry-utility-classes`、`revit-grid-curve-creation`

---

## D

### Dimension（尺寸标注）
- 中文译名：尺寸标注
- 一句话定义：注释类图元，识别与创建需区分 View-specific 与 Reference-based 两种类型。
- 相关 skill：`revit-dimension-identification`、`revit-element-six-groups`

### Document（文档）
- 中文译名：文档（数据层）
- 一句话定义：文件级数据对象，是外部命令的第一个参数、一切 API 操作的根，与 UI 层严格分离。
- 相关 skill：`revit-document-vs-uidocument`、`revit-document-function-map`

### DWG / DXF 导出（Export）
- 中文译名：CAD 格式导出
- 一句话定义：Document.Export 输出 DWG/DXF 等格式，图层映射与线型经预定义导出表（ExportTables）定制。
- 相关 skill：`revit-dwg-export-tables`、`revit-document-export-formats`

---

## E

### Edge（边缘）
- 中文译名：边缘
- 一句话定义：面的边界曲线段，参数化固定 0–1，可按面方向展开为同向曲线。
- 相关 skill：`revit-face-edge-loop-traversal`

### EdgeLoop（边缘环）
- 中文译名：边缘环
- 一句话定义：Face 上的一条封闭边界循环；多个环即"带洞的面"，天然区分外环与内环。
- 相关 skill：`revit-face-edge-loop-traversal`、`revit-opening-boundary-reading`

### Element（图元）
- 中文译名：图元
- 一句话定义：Revit 模型的基本对象，按功能六组（Model/Sketch/View/Group/Annotation/Information）预判 API 行为。
- 相关 skill：`revit-element-six-groups`、`revit-element-retrieval-four-entries`

### ElementClassFilter / ElementCategoryFilter（类/类别过滤器）
- 中文译名：快速过滤器
- 一句话定义：按 .NET 类型或 Category 缩小候选集的快速过滤器，应先用它们给慢速过滤器减负。
- 相关 skill：`revit-filter-quick-slow-logical`、`revit-updater-registration-triggers`

### ElementId（图元 ID）
- 中文译名：图元 ID
- 一句话定义：文档内唯一的整数标识，会话内传递与检索最快，比较应使用 IntegerValue。
- 相关 skill：`revit-elementid-vs-uniqueid`、`revit-id-vs-uid-decision`

### ElementIntersectsElementFilter / ElementIntersectsSolidFilter（相交过滤器）
- 中文译名：图元相交过滤器
- 一句话定义：几何级精确相交判断（碰撞/干扰检查）的慢速过滤器，均不可反转。
- 相关 skill：`revit-intersects-filter-decisions`、`revit-boundingbox-filter-tradeoffs`

### EnableWorksharing（启用工作共享）
- 中文译名：启用工作共享
- 一句话定义：以编程方式把项目转为工作共享模型，一旦启用不可逆转，须提前确认。
- 相关 skill：`revit-enableworksharing-irreversible`、`revit-worksharing-api-decision-framework`

### Entity（条目）
- 中文译名：存储条目
- 一句话定义：按 Schema 创建的数据条目，字段赋值后经 SetEntity 挂到图元上。
- 相关 skill：`revit-extensible-storage-pipeline`、`revit-data-storage-paths`

### ExternalEvent / IExternalEventHandler（外部事件）
- 中文译名：外部事件
- 一句话定义：把非主线程请求投递到主线程合法执行的机制，非模态窗口调 API 的唯一正路。
- 相关 skill：`revit-external-events-nonmodal-dialog`、`revit-transaction-off-thread`、`revit-transaction-thread-context`

### ExternalFileReference（外部文件引用）
- 中文译名：外部文件引用
- 一句话定义：描述项目引用的外部文件（链接/贴花等）路径与类型的非图元容器，配合 ExternalFileUtils 查询。
- 相关 skill：`revit-external-file-reference-utils`、`revit-link-decision-flow`

### ExtensibleStorage（可扩展存储）
- 中文译名：可扩展存储
- 一句话定义：在图元上存结构化、可权限控制的隐藏数据机制，走 Schema→Entity 六步流水线。
- 相关 skill：`revit-extensible-storage-pipeline`、`revit-data-storage-paths`

---

## F

### Face（面）
- 中文译名：面
- 一句话定义：Solid 表面的组成元素，边界由 EdgeLoops 描述，面积可直接读取。
- 相关 skill：`revit-face-edge-loop-traversal`、`revit-element-geometry-extraction`

### FailureDefinition（故障定义）
- 中文译名：故障定义
- 一句话定义：自定义故障的注册单元（ID/严重级别/描述），OnStartup 注册，运行时 PostFailure 发布、UnpostFailure 撤回；内置故障走 BuiltInFailures。
- 相关 skill：`revit-failure-definition-registration`、`revit-failures-processing-steps`

### FailureHandlingOptions（故障处理选项）
- 中文译名：故障处理选项
- 一句话定义：事务级故障配置对象，挂预处理器/设延迟等信息，必须 Get→修改→Set 写回才生效。
- 相关 skill：`revit-failure-handling-options`、`revit-failure-options-get-set`

### FailuresAccessor（故障访问器）
- 中文译名：故障访问器
- 一句话定义：故障处理各步骤的公共入参，负责读取、删除、解决当前事务的故障消息。
- 相关 skill：`revit-failures-accessor`、`revit-failures-processing-steps`

### FailureProcessingResult（故障处理结果码）
- 中文译名：故障处理结果
- 一句话定义：故障处理器返回的枚举，控制引擎下一步是提交、回滚还是继续处理。
- 相关 skill：`revit-failures-accessor`、`revit-failures-processor-global`

### FailureSeverity（故障严重级别）
- 中文译名：故障严重级别
- 一句话定义：故障的三级严重度：Warning、Error、DocumentCorruption，级别决定默认处置方式。
- 相关 skill：`revit-failure-severity`、`revit-failure-definition-registration`

### Family（族）
- 中文译名：族
- 一句话定义：某类别下定义几何与参数的"类文件"，是 Family→FamilySymbol→FamilyInstance 三层的中间层，分系统族与构件族。
- 相关 skill：`revit-family-three-layer-model`、`revit-system-vs-component-family`

### FamilyInstance（族实例）
- 中文译名：族实例
- 一句话定义：按 Symbol 放置到项目中的具体图元，有位置、宿主关系与实例参数。
- 相关 skill：`revit-category-family-symbol-instance`、`revit-instance-host-subcomponent-navigation`、`revit-instance-flip-state-check`

### FamilyItemFactory（族图元工厂）
- 中文译名：族图元创建工厂
- 一句话定义：族文档专属的形状创建入口（NewExtrusion/NewSweep 等），与项目文档创建体系分离。
- 相关 skill：`revit-familyitemfactory-shape-creation`、`revit-family-document-edit-paths`

### FamilyManager（族管理器）
- 中文译名：族管理器
- 一句话定义：族文档中管理族类型、族参数与公式的对象；类型值须在当前类型上下文设置。
- 相关 skill：`revit-familymanager-type-parameter`

### FamilySymbol（族符号）
- 中文译名：族类型（族符号）
- 一句话定义：族的一个具体类型定义，创建实例前必须先 Activate；界面"族类型"即此对象。
- 相关 skill：`revit-category-family-symbol-instance`、`revit-loadfamilysymbol-preference`、`revit-family-symbol-instance-navigation`

### FamilyElementVisibility（族图元可见性）
- 中文译名：族图元可见性
- 一句话定义：按"视图类型 × 详细程度"双轴控制族内图元显隐的对象，须经 SetVisibility 显式应用。
- 相关 skill：`revit-family-element-visibility`、`revit-family-document-edit-paths`

### FilteredElementCollector（过滤图元收集器）
- 中文译名：过滤图元收集器
- 一句话定义：按条件找图元的标准工具，构造函数（文档/ID 集/视图）决定检索范围，结果经 ToElements/ToElementIds 获取。
- 相关 skill：`revit-filtered-collector-three-steps`、`revit-collector-result-retrieval`、`revit-view-vs-document-collector`

### Form（形状）
- 中文译名：形状（概念设计）
- 一句话定义：体量/概念设计环境统一的形状类，New*Forms 系列创建，与族文件五具体类双轨不可混用。
- 相关 skill：`revit-conceptual-forms-type-selection`、`revit-referencepoint-curvebypoints`

---

## G

### GeometryElement（几何元素）
- 中文译名：几何元素
- 一句话定义：Element.Geometry 返回的几何容器，可迭代 GeometryObject 按类型分派处理。
- 相关 skill：`revit-element-geometry-extraction`

### GeometryInstance（几何实例）
- 中文译名：几何实例
- 一句话定义：嵌套族/实例化几何的"壳"对象，必须带 Transform 递归展开；GetSymbolGeometry 返回原件、GetInstanceGeometry 返回副本。
- 相关 skill：`revit-family-instance-no-geometryinstance`、`revit-symbol-vs-instance-geometry`、`revit-element-geometry-extraction`

### Geometry Options（几何选项）
- 中文译名：几何提取选项
- 一句话定义：控制几何提取返回内容的四属性配置：ComputeReferences、IncludeNonVisibleObjects、View、DetailLevel。
- 相关 skill：`revit-geometry-options-visibility`、`revit-element-geometry-extraction`

### Grid（轴网）
- 中文译名：轴网
- 一句话定义：定位轴线图元，Curve 属性区分 Line/Arc，创建须在水平面内，NewGrids 可批量创建。
- 相关 skill：`revit-grid-curve-creation`

---

## H

### Host（主体/宿主）
- 中文译名：主体（宿主）
- 一句话定义：承载族实例的图元（墙/楼板/天花板等），经 Instance.Host 获取，删除宿主连带删除实例。
- 相关 skill：`revit-instance-host-subcomponent-navigation`、`revit-newfamilyinstance-overload-decision`、`revit-foundation-host-deletion`

### HostObjectUtils（主体对象工具）
- 中文译名：主体对象工具
- 一句话定义：取墙/楼板等主体图元顶面/底面等侧面的静态辅助工具。
- 相关 skill：`revit-geometry-utility-classes`、`revit-element-geometry-extraction`

---

## I

### IFC / IExporterIFC / ExporterIFCRegistry（IFC 导出）
- 中文译名：IFC 自定义导出
- 一句话定义：实现 IExporterIFC 并在 OnStartup 注册即可接管全部 IFC 导出，一个会话只允许一个。
- 相关 skill：`revit-ifc-exporter-registration`、`revit-document-export-formats`

### IFailuresPreprocessor（故障预处理器）
- 中文译名：故障预处理器
- 一句话定义：事务级故障预处理接口，每事务至多一个、无默认，故障解决过程中最先获得控制。
- 相关 skill：`revit-failures-preprocessor`

### IPointCloudEngine（点云引擎）
- 中文译名：点云引擎
- 一句话定义：自定义格式点云的解析引擎接口，经 PointCloudEngineRegistry 注册后供 Revit 标准访问。
- 相关 skill：`revit-point-cloud-api`、`revit-point-cloud-filter`

### ISelectionFilter（选择过滤器）
- 中文译名：选择过滤器
- 一句话定义：限定用户拾取范围的接口，AllowElement 管图元整体、AllowReference 管面/边等子对象。
- 相关 skill：`revit-selection-pickobject-filter`

### IUpdater / DMU（更新器）
- 中文译名：动态模型更新器
- 一句话定义：模型变更时自动执行的回调机制；注册 + AddTrigger 双维度触发，Execute 借用外层事务、早于 DocumentChanged。
- 相关 skill：`revit-iupdater-execute-transaction-rules`、`revit-updater-registration-triggers`、`revit-updater-change-management`、`revit-updater-failure-modes`

### IExternalCommand（外部命令）
- 中文译名：外部命令
- 一句话定义：插件命令入口接口，Execute 在命令上下文由 Revit 调用，返回结果决定撤销行为。
- 相关 skill：`revit-external-command-entry`、`revit-execute-try-catch-message`、`revit-command-result-undo`

### IExternalApplication / IExternalDBApplication（外部应用）
- 中文译名：外接应用程序
- 一句话定义：随 Revit 启停的插件宿主接口，OnStartup 注册事件/UI、OnShutdown 注销；DB 版无 UI 语境。
- 相关 skill：`revit-plugin-entry-types`、`revit-command-app-loading-timing`、`revit-startup-shutdown-events`、`revit-event-registration-two-steps`

---

## L

### LoadFamily / LoadFamilySymbol（加载族）
- 中文译名：加载族/族符号
- 一句话定义：单类型放置优先 LoadFamilySymbol（更快更省内存）；需全部类型才用 LoadFamily。
- 相关 skill：`revit-loadfamilysymbol-preference`、`revit-family-document-edit-paths`

### Location / LocationCurve / LocationPoint（位置对象）
- 中文译名：定位（点/线）
- 一句话定义：图元的定位信息对象，分点（Position+Rotation）与线（Curve）两种形态；取驱动线不必进几何树。
- 相关 skill：`revit-location-type-decision`、`revit-instance-rotate-location-type`、`revit-element-geometry-extraction`

---

## M

### Material（材质）
- 中文译名：材质
- 一句话定义：图元外观与物理属性载体，经属性集（PropertySetElement）的 ElementId 关联；族文档创建入口受限。
- 相关 skill：`revit-material-access-creation`、`revit-material-retrieval-fallback`

### MEPSystem（MEP 系统）
- 中文译名：MEP 系统
- 一句话定义：管道/机械/电气系统的承载对象，由基础设备连接件+成员集+系统类型三要素创建。
- 相关 skill：`revit-mep-connector-system-topology`

### ModelCurve / ModelLine（模型线）
- 中文译名：模型线/模型曲线
- 一句话定义：挂在 SketchPlane 上的模型级曲线；创建形状时会被"消耗"，可经 ChangeToReferenceLine 转参照线。
- 相关 skill：`revit-sketch-sketchplane-model`、`revit-model-vs-reference-line`

### MoveElement / Rotate（移动与旋转）
- 中文译名：移动/旋转图元
- 一句话定义：ElementTransformUtils 的图元编辑方法；Pinned=true 时被拒绝，移动注意 Z 坐标陷阱。
- 相关 skill：`revit-element-move-decision`、`revit-element-transform-utils`、`revit-moveelement-z-coordinate-trap`

---

## N

### NewFamilyInstance（新建族实例）
- 中文译名：新建族实例
- 一句话定义：放置族实例的核心入口，12 个重载按宿主/标高/面/参照等位置特征选择；批量放置走 NewFamilyInstances。
- 相关 skill：`revit-newfamilyinstance-overload-decision`、`revit-newfamilyinstances-batch-create`

### NewOpening / Opening（洞口）
- 中文译名：洞口
- 一句话定义：在墙/板/竖井等主体上开洞的图元，重载按边界曲线与参照方式选择；边界经 BoundaryRect/EdgeLoops 读取。
- 相关 skill：`revit-new-opening-overloads`、`revit-new-opening-constraints`、`revit-opening-boundary-reading`

---

## P

### Parameter（参数）
- 中文译名：参数
- 一句话定义：图元携带的数据字段，按名称/内置枚举/GUID 定位，写入前须判空与只读。
- 相关 skill：`revit-parameter-index-lookup`、`revit-builtin-parameter-semantics`、`revit-parameter-storage-types`

### ParameterType（参数语义类型）
- 中文译名：参数语义类型
- 一句话定义：描述参数意义的枚举（Length/YesNo/Text 等 15 种），决定 UI 显示与单位格式化。
- 相关 skill：`revit-parameter-storage-types`
- ⚠️ 版本提示：Revit 2014 的 ParameterType 枚举在新版已被 ForgeTypeId 定义体系取代。

### PathType（链接路径类型）
- 中文译名：链接路径类型
- 一句话定义：外部引用路径的存储方式三选一：Relative、Absolute、Server；Server 路径（ServerPath）有专门坑。
- 相关 skill：`revit-link-decision-flow`、`revit-serverpath-avoid`

### PerformanceAdviser（性能顾问）
- 中文译名：性能顾问
- 一句话定义：内置性能检查框架，兼规则注册中心与执行引擎，AddRule/DeleteRule 生命周期须配对。
- 相关 skill：`revit-performance-adviser-rules`、`revit-performance-adviser-rule-interface`

### PickObject / Selection（拾取与选集）
- 中文译名：拾取/选择集
- 一句话定义：提示用户点选对象的交互 API（单/多/框选/取点）；Selection.Elements 读当前选集，必须运行在 UI 上下文。
- 相关 skill：`revit-selection-pickobject-filter`

### PointCloudType / PointCloudInstance（点云）
- 中文译名：点云类型/实例
- 一句话定义：两步创建点云：先建类型（指定引擎标识）再建实例；取点用 GetPoints 配过滤器。
- 相关 skill：`revit-point-cloud-api`、`revit-point-cloud-filter`

### PostCommand（发布命令）
- 中文译名：发布命令
- 一句话定义：把 Revit 内置命令塞入消息队列异步执行的机制；同一时间只允许一个待发布命令。
- 相关 skill：`revit-postcommand-single-limit`

---

## Q

### QuickFilter / SlowFilter / LogicalFilter（快慢与逻辑过滤器）
- 中文译名：快速/慢速/逻辑过滤器
- 一句话定义：快慢两档成本不同的过滤器体系，配 AND/OR/NOT 逻辑组合；慢速过滤器必须先被快速缩圈。
- 相关 skill：`revit-filter-quick-slow-logical`、`revit-intersects-filter-decisions`

---

## R

### Reference（参照）
- 中文译名：参照
- 一句话定义：标识"几何表示树路径"的稳定句柄，分端点/曲线/面/边四类；可序列化为字符串跨会话还原。
- 相关 skill：`revit-reference-stable-handle`、`revit-selection-pickobject-filter`

### ReferenceIntersector（参照求交器）
- 中文译名：参照求交器
- 一句话定义：光线投影查询工具：从起点沿方向发射光线，返回命中参照集合或最近一个，仅限三维几何。
- 相关 skill：`revit-reference-intersector-raycast`

### ReferenceLine（参照线）
- 中文译名：参照线
- 一句话定义：形状创建后仍保留且能驱动形状更新的线；API 无独立类，由模型线转换而来。
- 相关 skill：`revit-model-vs-reference-line`

### ReferencePoint（参照点）
- 中文译名：参照点
- 一句话定义：概念设计环境的参数化驱动源，用 PointElementReference 子类吸附到边/面/平面/交点。
- 相关 skill：`revit-referencepoint-curvebypoints`

### Regenerate（重新生成）
- 中文译名：重新生成
- 一句话定义：提交几何/参数求值的调用，时机与次数直接影响性能，失败会触发回滚。
- 相关 skill：`revit-regenerate-geometry-timing`、`revit-regenerate-cost`、`revit-regenerate-failure-rollback`

### RevitLinkType / RevitLinkInstance（Revit 链接）
- 中文译名：Revit 链接
- 一句话定义：链接 rvt 的两层对象：先建 RevitLinkType 再建 RevitLinkInstance；嵌套链接经 GetChildIds 遍历。
- 相关 skill：`revit-link-decision-flow`、`revit-link-reference-conversion`、`revit-nested-link-unload`

### Ribbon（功能区）
- 中文译名：功能区
- 一句话定义：插件 UI 的宿主容器（Tab/Panel/控件），按钮与面板的类型和布局规则均受 API 约束。
- 相关 skill：`revit-ribbon-control-types`、`revit-ribbon-panel-layout`

### Room（房间）
- 中文译名：房间
- 一句话定义：由边界围合的非图形空间图元；门的 To/From Room 语义区分进出方向，体积须显式启用。
- 相关 skill：`revit-room-creation-boundary`、`revit-to-from-room-semantics`、`revit-room-volume-enable`

---

## S

### Schema / Field / SchemaBuilder（架构与字段）
- 中文译名：存储架构/字段/构建器
- 一句话定义：可扩展存储的数据结构定义及其字段；SchemaBuilder Finish 之后不可再改，带单位字段必须设 DisplayUnit。
- 相关 skill：`revit-schema-immutable-after-finish`、`revit-schema-fieldbuilder-one-shot`、`revit-extensible-storage-pipeline`

### SharedParameter（共享参数）
- 中文译名：共享参数
- 一句话定义：跨项目/族复用的参数定义，存于外部共享参数文件；用前必须先设路径再打开，GUID 须校验。
- 相关 skill：`revit-shared-parameter-file-init`、`revit-shared-parameter-guid-check`、`revit-shared-parameter-file-swap`

### Sketch / SketchPlane（草图与草图平面）
- 中文译名：草图/草图平面
- 一句话定义：主体图元的瞬态轮廓及其所在工作平面；换平面须 SetPlaneAndCurve 一次同步改，草图不可经 Document.Elements 枚举。
- 相关 skill：`revit-sketch-sketchplane-model`、`revit-grid-curve-creation`
- ⚠️ 版本提示：原书提及的 Document.Elements 枚举在新版已移除，一律改用 FilteredElementCollector。

### Solid（实体）
- 中文译名：实体
- 一句话定义：三维封闭体几何对象，Faces 与 Edges 承载全部边界信息，也是相交过滤器的输入。
- 相关 skill：`revit-element-geometry-extraction`、`revit-intersects-filter-decisions`

### SpatialFieldManager（空间场管理器）
- 中文译名：空间场管理器
- 一句话定义：分析结果（能量/结构等）在图元上做着色可视化的管理器，结果刷新有专门机制。
- 相关 skill：`revit-analysis-results-refresh`

### Stairs（楼梯）
- 中文译名：楼梯
- 一句话定义：楼梯对象模型（Stairs/Run/Landing/Landing），编辑须在专门编辑作用域内做隔离。
- 相关 skill：`revit-stairs-object-model`、`revit-stairs-edit-scope-isolation`

### StorageType（存储类型）
- 中文译名：存储类型
- 一句话定义：参数底层存储格式的五种枚举（String/ElementId/Double/Integer/None），决定 Get/Set 方法对。
- 相关 skill：`revit-parameter-storage-types`

### SubTransaction（子事务）
- 中文译名：子事务
- 一句话定义：只能开在事务内部的局部修改单元，可单独回滚而不影响外层事务的其他修改。
- 相关 skill：`revit-transaction-hierarchy`、`revit-iupdater-execute-transaction-rules`

### SynchronizeWithCentral（与中心同步）
- 中文译名：与中心同步
- 一句话定义：双向同步 API：先重载中心变更再保存回中心；需临时锁定中心模型，只读拉取走 ReloadLatest。
- 相关 skill：`revit-synchronize-with-central`

---

## T

### TaskDialog（任务对话框）
- 中文译名：任务对话框
- 一句话定义：Revit 统一模态提示框，固定 12 元素按标准顺序排列，适合提示/确认/简单选择。
- 相关 skill：`revit-taskdialog-vs-dialog`、`revit-taskdialog-component-rules`

### Transform（变换）
- 中文译名：变换
- 一句话定义：几何的平移/旋转矩阵对象，展开实例几何、符号几何转项目坐标都必须经它完成。
- 相关 skill：`revit-element-transform-utils`、`revit-symbol-vs-instance-geometry`

### Transaction（事务）
- 中文译名：事务
- 一句话定义：模型修改的最小上下文，一次只能开一个、禁止嵌套；命令的事务行为分 Manual/Automatic/ReadOnly 三种模式。
- 相关 skill：`revit-transaction-hierarchy`、`revit-transaction-mode-selection`、`revit-iupdater-execute-transaction-rules`

### TransactionGroup（事务组）
- 中文译名：事务组
- 一句话定义：把多个独立事务组装成组的容器，支持整组回滚或 Assimilate 融合为单个撤销项。
- 相关 skill：`revit-transaction-hierarchy`

### TransmissionData（传输数据）
- 中文译名：传输数据
- 一句话定义：离线读写外部引用路径信息的数据结构，不开文档即可批量改链接路径。
- 相关 skill：`revit-transmissiondata-offline`

---

## U

### UIDocument（UI 文档）
- 中文译名：UI 文档
- 一句话定义：界面级文档对象，承载选择拾取、激活视图等交互；无 UI 语境（DB 应用）不可用。
- 相关 skill：`revit-document-vs-uidocument`、`revit-selection-pickobject-filter`

### UniqueId（唯一 ID）
- 中文译名：唯一标识
- 一句话定义：图元跨文档/跨会话全局唯一的 GUID 标识，任何要离开当前会话的持久化引用都应存它。
- 相关 skill：`revit-elementid-vs-uniqueid`、`revit-id-vs-uid-decision`

### UnitUtils（单位工具）
- 中文译名：单位转换工具
- 一句话定义：内部单位（长度英尺/角度弧度）与显示单位间的转换工具；显示与存储间永远显式换算。
- 相关 skill：`revit-internal-units-conversion`

---

## V

### View（视图）
- 中文译名：视图
- 一句话定义：视图本身也是图元；分类有三条并行路径：ViewType 枚举、派生类类型、ViewFamilyType。
- 相关 skill：`revit-view-classification-paths`、`revit-view-vs-document-collector`

### View3D（三维视图）
- 中文译名：三维视图
- 一句话定义：CreatePerspective/CreateIsometric 创建的三维视图，观察方向经 ViewOrientation3D 设置，剖面框/裁剪框控范围。
- 相关 skill：`revit-3d-view-creation-flow`、`revit-printable-view-check`

### ViewPlan（平面视图）
- 中文译名：平面视图
- 一句话定义：平/立/剖等二维视图的创建入口族；标高创建后不会自动生成关联平面视图。
- 相关 skill：`revit-2d-view-creation-paths`、`revit-level-no-auto-view`

### ViewFamilyType（视图族类型）
- 中文译名：视图族类型
- 一句话定义：视图的类型对象（ViewFamily 枚举），是视图创建 API 唯一认的分类路径。
- 相关 skill：`revit-view-classification-paths`、`revit-3d-view-creation-flow`

### ViewSchedule / Schedule（明细表）
- 中文译名：明细表
- 一句话定义：派生自 View 的表格视图，9 种工厂方法创建，字段/分组/过滤经 ScheduleDefinition 配置。
- 相关 skill：`revit-schedule-creation-flow`

### ViewSheet（图纸）
- 中文译名：图纸
- 一句话定义：出图用容器视图，可经 CreateSheetList 列举；打印前须校验视图可打印性。
- 相关 skill：`revit-schedule-creation-flow`、`revit-printable-view-check`

### VisibilityMode（可见性模式）
- 中文译名：命令可见性模式
- 一句话定义：外部命令的静态可见性：项目/族/无文档三态，叠加专业（Discipline）过滤控制按钮何时出现。
- 相关 skill：`revit-command-visibility-mode`、`revit-command-availability`

---

## W

### Wall / WallLocationLine（墙与定位线）
- 中文译名：墙/墙定位线
- 一句话定义：基础建模图元，Wall.Create 重载由输入形态决定；定位线决定放置基准并联动复合结构偏移。
- 相关 skill：`revit-wall-create-overloads`、`revit-wall-location-line`、`revit-wall-structural-usage-rules`

### Workset / Worksharing（工作集/工作共享）
- 中文译名：工作集/工作共享
- 一句话定义：多用户协作模式：中心模型+本地副本两层存储，编辑前检出、完成后放弃所有权。
- 相关 skill：`revit-worksharing-api-decision-framework`、`revit-open-workshared-file`、`revit-synchronize-with-central`

---

## X

### XYZ（三维坐标点）
- 中文译名：三维坐标点
- 一句话定义：Revit 三维坐标结构体，绝大多数字面量坐标输入用它；配合 Plane 等纯几何工具类使用。
- 相关 skill：`revit-geometry-utility-classes`、`revit-wall-create-overloads`

---

## 中文术语速查（按拼音）

- 边界框过滤器 → BoundingBox 过滤器族；边界轮廓 → Outline
- 标注/注释图元 → Annotation（见 Element 功能六组）
- 部件/零件/零件分割器 → Assembly / Part / PartMaker
- 参数 → Parameter；参数绑定 → Binding；参数语义类型 → ParameterType；存储类型 → StorageType；内建参数 → BuiltInParameter
- 草图/草图平面 → Sketch / SketchPlane
- 类别 → Category；类/类别过滤器 → ElementClassFilter / ElementCategoryFilter
- 族/族符号/族实例 → Family / FamilySymbol / FamilyInstance
- 族管理器 → FamilyManager；族图元工厂 → FamilyItemFactory；族图元可见性 → FamilyElementVisibility
- 事务/子事务/事务组 → Transaction / SubTransaction / TransactionGroup；事务模式 → TransactionMode；事务融合 → Assimilate
- 分析模型 → AnalyticalModel；分析链接 → AnalyticalLink
- 故障定义/访问器/预处理器/结果码/严重级别 → FailureDefinition / FailuresAccessor / IFailuresPreprocessor / FailureProcessingResult / FailureSeverity
- 检出/放弃所有权 → CheckoutElements / WorksharingUtils
- 洞口 → Opening；复合结构 → CompoundStructure；边界矩形 → BoundaryRect
- 可扩展存储 → ExtensibleStorage；架构/字段/构建器 → Schema / Field / SchemaBuilder；条目 → Entity
- 链接 → RevitLinkType / RevitLinkInstance；外部文件引用 → ExternalFileReference；路径类型 → PathType
- 明细表 → ViewSchedule；模型线 → ModelCurve；参照线 → ReferenceLine；参照点 → ReferencePoint
- 变换 → Transform；几何实例 → GeometryInstance；几何元素 → GeometryElement
- 快速/慢速/逻辑过滤器 → QuickFilter / SlowFilter / LogicalFilter
- 平面视图 → ViewPlan；三维视图 → View3D；视图族类型 → ViewFamilyType；图纸 → ViewSheet
- 外部事件 → ExternalEvent；发布命令 → PostCommand；可见性模式 → VisibilityMode
- 唯一标识 → UniqueId；图元 ID → ElementId；图元 → Element
- 与中心同步 → SynchronizeWithCentral；重载最新 → ReloadLatest；工作集/工作共享 → Workset / Worksharing
- 性能顾问 → PerformanceAdviser；任务对话框 → TaskDialog；空间场管理器 → SpatialFieldManager
- 参照 → Reference；参照求交器 → ReferenceIntersector；选择过滤器 → ISelectionFilter；拾取 → PickObject
- 非模态对话框 → ExternalEvent 框架
- 轴网 → Grid；圆弧 → Arc；三维坐标点 → XYZ
- 梁系统 → BeamSystem；楼梯 → Stairs；房间 → Room；材质 → Material
- 检索四法 → 见 `revit-element-retrieval-four-entries`；功能六组 → 见 `revit-element-six-groups`
- 主体（宿主）→ Host；主体对象工具 → HostObjectUtils
- 移动/旋转 → MoveElement / Rotate；固定 → Pinned
- 点云 → PointCloudType / PointCloudInstance；点云引擎 → IPointCloudEngine
- DWG 导出 → DWG / DXF 导出；IFC 导出 → IFC / IExporterIFC / ExporterIFCRegistry
- 复制图元 → CopyElements；尺寸标注 → Dimension
- 外部命令/外部应用 → IExternalCommand / IExternalApplication
- 单位转换 → UnitUtils；内部单位 → InternalUnits
- 传输数据 → TransmissionData；启用工作共享 → EnableWorksharing

---

## 审计信息

- 词条总数：110
- 覆盖领域：对象模型、过滤检索、事务与故障处理、几何、族、参数与数据存储、事件与更新器、视图、链接与导出、工作共享、MEP/结构/点云/楼梯/材质等专题
- 挂接 skill 数：149 个（159 个 skill 中覆盖最深的部分；slug 均经 `_stage2/skill-map.yaml` 核对，无编造）
- 版本提示：AnalyticalModel、ParameterType、Sketch（Document.Elements）三条含 Revit 2014 时代背景标注
- 蒸馏时间：2026-08-28
- 备注：本文件为两位队友版本的合并版（多挂接 + 版本提示 + 中文索引骨架，并入对方独有词条），slug 校验脚本复核通过
