---
name: bannerlord-character-mod
description: 把别的游戏/来源的角色模型做成《骑马与砍杀2：霸主》的装备（皮套）mod：提取原始蒙皮权重、405→27 骨骼映射、姿态重定向、.tpac 资产打包（网格/贴图/材质）、装备 XML、离线与实机验证，以及移植后各类显示问题的定位（颜色错位、纯黑/不透明白片、脸型扭曲、手脚错位、站姿怪异、原版身体露出来）。
whenToUse: 用户要把某个角色模型（FBX/OBJ/.blend/游戏提取件）做成骑砍2 的装备/皮套/换装 mod；或要排查已移植模型在游戏里的显示问题（颜色错位、镜片/头发/睫毛不透明、脸型扭曲、拇指或手肘异常、站姿怪、原版脚露出来）。也可用于给骑砍2 手写 .tpac 资产包。
user-invocable: true
---

# 骑砍2 角色皮套 mod 制作 Skill

把外部角色模型移植成 Bannerlord 装备件。**整条链路已跑通过一次完整工程**，可直接复用：

| 资源 | 位置 |
|---|---|
| 参考实现（含可复现管线与全部实测数字） | `D:\dsh-mb-mod`（公开：https://github.com/tridkx/bannerlord-valerie-harmon-rebuild ） |
| `.tpac` 工具集（打包器 + Python 工具 + 格式文档） | `D:\mb-tools`（公开：https://github.com/tridkx/bannerlord-tpac-toolkit ） |
| 现成对照 mod（官方 Modding Kit 产出、显示正常） | `<游戏>\Modules\LVBU and DIAOCHAN\` |
| 游戏 | `D:\SteamLibrary\steamapps\common\Mount & Blade II Bannerlord` |

> **先看成品再动手**：拿参考 mod 的 `pack0.tpac` 用 `mbtool matall` / `mbtool mesh` / `mbtool mat`
> dump 一遍，就知道"正常长什么样"。移植里绝大多数参数问题都是靠这个差集定位的。

---

## 0. 交付物形态（先建立正确预期）

一个装备 mod = **三个文件**：

```
Modules\<ModName>\
    SubModule.xml                 模块声明（Id/Name/依赖/挂 items.xml）
    ModuleData\items.xml          物品定义（mesh 名 → 物品，槽位，护甲值）
    AssetPackages\pack0.tpac      资产包（网格 + 贴图 + 材质，全部在这里）
```

不需要 C# DLL、不需要 Modding Kit 工程 —— 只要你有一个能写 `.tpac` 的工具（见 §4）。

---

## 1. 先体检源模型（决定难度与路线）

**这一步决定后面所有工作量**，别跳。检查项：

| 检查 | 为什么关键 |
|---|---|
| 有没有骨骼？ | 没骨骼 = 要自己摆姿势、对齐骨架 |
| **有没有蒙皮权重？权重在哪个文件里？** | 最关键的变量，见 §2 |
| 骨架骨名/骨数 | 决定映射表怎么写 |
| 材质分组与贴图通道 | 决定材质与贴图怎么拆 |
| 面数 / 部件数 / LOD | 决定性能与是否拆件 |

常用脚本：`D:\dsh-mb-mod\tools\intake_report.py`（面数/材质/UV/缝合片/骨架一览）。

---

## 2. ★ 权重：优先用源模型的**原始权重**，不要自己算

**本 skill 最重要的一条经验。**

- OBJ **不携带**蒙皮权重。如果源模型是"从游戏里导出的 OBJ"，很容易以为没有权重 ——
  但**原始 `.mesh`（或 FBX/glTF）里通常仍在**，只是导出格式丢了。
- 实测：RE Engine 的 `.mesh` 里 100% 顶点带权、每顶点 8 槽、精确归一（每顶点权重和 = 1.0000）。
- 用 `tools/export_weights.py` 解析原始网格并**与 OBJ 顶点逐组对齐**（实测最大偏差 5e-07 = float32 打印精度）。
- 自己按"到骨段距离"算权重，在**面部微骨（几百根）与手指**这些地方必然出错 —— 上一版
  的"脸型扭曲 + 大拇指弯折"就是自算权重的产物。

权重索引的坐标系要确认：RE 的权重索引指向 `skeleton.json` 的 `weightedBones` 表（**不是全骨表**），
用"索引范围 < len(weightedBones)"校验即可。

---

## 3. 骨骼映射：405 根 → 27 根，规则要能审计

Bannerlord 人形骨架 = 28 个引擎索引（**名字末尾的 `_N` 才是引擎索引**，FBX 导入后的列表下标不同，
中间夹着 `_notused` 骨）。映射规则按优先级：

| 规则 | 说明 |
|---|---|
| 主骨显式表 | 24 根主骨 → 引擎索引，手工核定 |
| ★ 面部/头发 → **head(13)** | 靠"骨链位置"会把它们判到 **neck(12)** —— 脸大部分在 head 关节(1.573m) **之下**、离 neck(1.466m) 更近。Bannerlord 的头是单根刚体骨，面部微骨必须显式归 head |
| ★ 手指 → **hand(19/26)** | 散到前臂扭曲骨(18/25) 会让同一只手被两根骨撕开 = 实测的"大拇指弯折" |
| 扭曲骨按段 | `*_humerus_twist*`→16/23、`*_radius_twist*`→18/25；腿无扭曲骨 → 归股/胫 |
| 背包/背带/布料 → 躯干链 | 否则背包跟着手臂甩 |
| 其余 → 骨链分数匹配 | 算该骨在最近骨链上的归一化位置 s，取目标骨链上分数最接近的骨。**分数与姿势无关**，因此不受"源绑定姿势 vs BL A-pose"影响 |

**交叉验证技巧**：身体网格的 head 权重应当是 0%（身体不含头），脸≈99%、头发=100% ——
三件互相印证，能立刻发现映射错位。权重的 8→4 槽归并几乎无损（实测脸 0.0001%，因为微骨归并到同一根骨）。

---

## 4. 姿态重定向（把源绑定姿势搬进 BL 的 A-pose）

逐骨变换（引擎空间：Z 上、+Y 前、+X 右）：

```
W_b(v) = u_b + s · Rot_b · (v − t_b)
  s, R    全局相似变换（Umeyama，只用躯干+腿拟合，**别把前伸的手臂算进去**）
  t_b/u_b 源骨/BL 骨的 head
  Rot_b = M_b · R，M_b = 把源骨轴（经 R 旋转）转到 BL 骨轴的最小旋转
```

要点：

1. **骨轴必须用"关节位置差"推导**，不能从骨骼矩阵取（源工程有的骨轴在矩阵 X 列、有的在 Y 列；
   Blender 导入的 BL 骨 Y 轴与子骨方向差 90°）。用一张手工核定的"骨链下一关节"表。
2. **判断手臂方案要量化，别靠感觉**：量"腕部落点 vs BL 手关节"的偏差。
   - 逐骨最小旋转 + **链式落点**（`u_w = u_r + s·Rot_r·(t_w − t_r)`）= 偏差 3.9cm（方向正确）
   - 整条手臂刚性 = 偏差 21.8cm 且**朝前 16cm**（双手一直端着）→ 否掉
   - 那 3.9cm 的来源是长度差（源前臂×s 22.9cm vs BL 26.8cm），不是错误
3. 手腕"共用前臂旋转 + 落点沿链推出"能同时消掉扇叶与接缝。
4. 姿态微调旋钮（都绕关节转，可归零）：
   - `SHOULDER_FWD_DEG`：绕**锁骨根部**只转肩+手臂（躯干不动）→ 治"肩臂太靠后"
   - 不要用"整体前倾躯干"来治肩臂靠后 —— 那会把肚子/胸一起推出去（实测被用户否掉）
   - `HEAD_SCALE / HEAD_LIFT`：头偏小/脖子看不见
   - X 方向分区收窄（躯干/腿）治"体型太壮"；**手臂保持 1.0**，否则手偏离 hand 骨、持械错位
5. **剔除隐藏"缝合片"**：源模型常带窄长三角（静止姿势最长边 >6cm，实测只占 248/85028）。
   判据要在**缩放/比例修正之前**算 —— 否则把缩放后的正常三角一并误删（实测头发多删 1400+ 面）。

---

## 5. `.tpac` 打包（模板克隆）

所有资产都从**已有包里的模板资产克隆**（模板提供 shader、顶点布局等难构造字段），只换数据/名字/GUID：

```bash
mbtool build spec.json     # spec 里 meshes / materials / textures / templates
```

要点：

- **`.mgeo` 中间格式**（见 `D:\mb-tools\README.md`），含 4 组骨索引/权重
- **UV 存 `(u, 1−v)`**：源模型多为 D3D 习惯（v=0 在贴图顶部），`.bc` 按 PNG 行序写盘，
  引擎采样会整体上下读反 → 颜色错位 + 法线错位（**凭空多褶皱**）。**必须在算切线之前翻**
- **`.bc` 必须带完整 mip 链**，否则打包器报 `blob too small for mip 1`
- 顶点/索引 ≥60000 时打包器自动拆子网格
- 材质模板从原版 `materials.tpac` 或参考 mod 取；半透明件见 §6

---

## 6. ★★ 贴图与透明：最容易翻车的一节

### 6.1 带 alpha 的贴图必须声明 `systemFlags=["has_alpha"]`

**引擎判断"这张贴图有没有 alpha 通道"看的是资产标志，不是文件里有没有 alpha 像素。**

没声明时的表现极具迷惑性：材质侧 `blendMode` / `alphaTest` / alpha 数值**怎么改都不生效**，
渲染结果**跟着贴图颜色走** —— 深色贴图 = 黑片、亮色贴图 = 白片。

```bash
mbtool tex  <pack> <贴图名> | grep systemFlags     # 自查：应为 [has_alpha]
```

对照：原版 `battania_dress_c_d`、参考 mod 的 `body_D` 都是 `[has_alpha]`。
手写打包器极易漏这一条（本工程为此折腾了 4 轮）。**spec 里给所有 DXT5/带 alpha 的贴图加上它。**

### 6.2 透明的两个条件（缺一不可）

1. **贴图带 alpha**（§6.1 的 `has_alpha` + 实际 alpha 像素）
2. **材质不能是 `no_alpha_blend`** —— 走透明管线：
   - `modulate`（modulator）：**相乘**语义，结果 = 背景 × 贴图RGB。所以镜片这类要给它**亮**贴图，
     否则背景被压黑
   - `alphaTest > 0` + `shaderMatFlags=[alpha_test]`：**镂空**（低于阈值整片丢弃）——
     源 alpha 若"镜身近 0、高光边缘 255"，配上阈值就得到"通透玻璃 + 边缘反光"
   - 原版真半透服饰 `khuzait_dress_b_alpha_mat` = `modulate` + `alphaTest=0.2745` + `alpha_test`
   - 相关 shaderFlag：`use_albedo_alpha`（显式启用 albedo 的 alpha）、`disable_vertex_color_alpha`
3. 换模板时**必须检查 `vertexLayoutFlags` 里有没有 `skinning`** —— 掉了它网格不跟骨骼动
   （打包日志里 `layout=[bumpmap]` 就是判据，应为 `[bumpmap,skinning]`）

### 6.3 自己生成的贴图：背景不能留大片纯黑

1024² 里只画两个小岛、其余全黑 → 小尺寸/远处时引擎采样**高 mip**，把内容与黑背景平均掉
（实测"眼睛纯黑"就是这个原因，而 Blender 近距渲染 mip0 看着正常，极易误判）。
**背景填成该内容自身的平均色**，并逐 mip 复核最暗值。

### 6.4 贴图通道语义（RE 系）

`_albm/_albmsc` 是基色（A 通道**可以**当透明度用，实测镜片区中位 10、高光边缘 255）、
`_nrmr` 是法线+粗糙度打包、`_atos` 是 alpha/SSS/遮蔽。
材质槽位编号在不同模板里语义不同（自己 dump 模板的槽位贴图名对照：`_d` 基色 / `_n` 法线 / `_s` 高光 / `_c` 色图）。

---

## 7. 装备 XML 与槽位

```xml
<Item id="vh_body" name="{=vh_body}名字" mesh="vh_body" Type="BodyArmor" weight="8" appearance="1.5">
  <ItemComponent><Armor body_armor="40" covers_body="true" covers_hands="true"
      has_gender_variations="false" modifier_group="cloth_unarmoured" material_type="Cloth"/></ItemComponent>
  <Flags UseTeamColor="false" Civilian="true"/>
</Item>
```

槽位经验：

- **头部槽（HeadArmor）一次只能穿一件** → 脸和头发必须放进**同一个 mesh**，否则戴上脸就没头发
- 原版角色的腿脚不会自动隐藏：鞋袜要做成 **LegArmor + `covers_legs="true"`**，
  否则原版的脚会从你的鞋子里露出来
- 一个 `<Item>` 里 `<Flags>` **只能有一个**（`maxOccurs=1`），多写会 schema 报错
- 物品栏搜索按**显示名**过滤，不是 id

---

## 8. 验证：三层，越早越省时间

| 层 | 手段 | 判据 |
|---|---|---|
| 资产正确性 | `mbtool metacheck` / `segcheck` | `checksum ok=N bad=0`、`metadata identical=N differ=0`、各段 `identical` |
| 形状/姿势 | Blender 无头渲染（正面/侧面/顶视 + 按权重自动取景的特写）；逐骨区域拟合相似变换看残差 | 残差 ~0 = 该区域刚性未被揉过（实测脸 0.07mm、头发 0.00mm、手 0.00mm） |
| 实机 | 进游戏看观感（UV/法线/透明/姿态只能这样判） | 见 §9 |

**渲染脚本用自己简化材质**，所以**判定不了 `.tpac` 材质模板相关问题**（透明、shader 效果）——
那些必须实机。反过来，形状问题（扭曲、错位）渲染就能看出来，别浪费实机轮次。

---

## 9. 症状 → 根因速查

| 症状 | 根因 |
|---|---|
| 颜色错位 / 袜子变深蓝 / 鞋色不对 / 衣服凭空多褶皱 | UV 的 V 轴没翻（或翻错时机，切线手性错） |
| 半透明件是**纯黑片** | 材质 `no_alpha_blend`，或（更常见）**贴图缺 `has_alpha`** → 深色贴图被当不透明 |
| 半透明件是**纯白片** | 同上，但贴图是亮色（颜色跟着贴图走 = 被当不透明） |
| 远处/小物件变黑，近看正常 | 自生成贴图的黑背景被高 mip 平均掉 |
| 脸型扭曲 | 用了自算权重（面部几百根微骨）→ 改用原始权重 + 微骨显式归 head |
| 拇指/手指弯折 | 手指骨散到前臂扭曲骨 → 全部归 hand(19/26) |
| 手肘向后弯、双手端着 | 手臂整链刚性 → 改逐骨 + 链式落点（量化判据见 §4.2） |
| 肩臂太靠后 | 绕**锁骨根部**转肩+臂（不是整体前倾躯干） |
| 挺胸/肚子凸 | 整体前倾躯干的副作用；只动肩 |
| 头偏小、脖子看不见 | 头缩放/抬升（绕颈关节） |
| 体型太壮 | 按骨权重分区做 X 收窄；手臂保持 1.0 |
| 原版脚露出来 | 缺 LegArmor + `covers_legs` |
| 头发/睫毛不透明、边缘发白 | 同"纯黑片"：贴图缺 `has_alpha`（镂空失效） |
| 网格不跟骨骼动 | 材质模板 `vertexLayoutFlags` 缺 `skinning` |
| 打包报 `blob too small for mip 1` | `.bc` 少了 mip 链 |

---

## 10. 方法论（比任何参数都值钱）

1. **要判断某个效果怎么做，先 dump 一个"已知能正常工作"的同类产物做参数差集。**
   参数空间靠推理试错会空转很多轮（本工程在镜片透明上白跑了 4 轮，最后是
   `mbtool matall` 对比参考 mod 才定位）。
2. **"像素数据"和"资产元数据"是两回事。** 反复验证 `.bc` 里的 alpha 字节，却从没验证贴图资产的
   `systemFlags` —— 后者才是引擎的判断依据。
3. **缓存键必须包含影响输出的全部参数。** 只比 mtime 的缓存会在改了编码参数后静默复用旧产物
   （实测：改了 alpha 参数但 `.bc` 被判"新鲜"，管线全绿、产物是旧的）。
4. **改完先量化再动手**：把"我以为"换成数字（落点偏差、残差、面数、mip 最暗值）。
5. **实机反馈是唯一的最终判据**，但要让每一轮实机都"能区分假设" ——
   例如用纯色贴图做二分（网格是否在渲染），而不是再猜一次参数。

---

## 11. 快速上手命令

```bash
export PYTHONUTF8=1 PYTHONIOENCODING=utf-8      # 中文输出不乱码
MB=/d/mb-tools/mbtool/bin/Release/net9.0/mbtool.exe      # 需先 dotnet build（见 mb-tools README）

$MB info   pack0.tpac                     # 资产统计
$MB matall pack0.tpac > all.tsv           # dump 全部材质参数（做差集用）
$MB mat    pack0.tpac <材质名>             # 单个材质：blendMode/alphaTest/flags/layer/槽位
$MB tex    pack0.tpac <贴图名>             # 单个贴图：尺寸/mip/格式/**systemFlags**
$MB mesh   pack0.tpac <网格名>             # 子网格 → 材质、包围盒、骨骼
$MB metacheck pack0.tpac                  # 校验和 + metadata 往返
$MB segcheck  pack0.tpac <资产名>          # 逐段字节比对（写入器是否保真）
```

完整工程（含全部脚本与实测数字）：见 `D:\dsh-mb-mod\PROGRESS.md` 与 README。
