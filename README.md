# dsh-skill-bannerlord-character-mod

> [!WARNING]
> **AI 生成声明 / AI-Generated Notice**
> 本项目（代码与文档）由 AI 辅助生成，仅供个人学习、参考与二次开发使用。
> 请在使用前自行审查代码的安全性，作者不对使用本项目造成的任何后果负责。
> This project (code & docs) was AI-assisted. Use at your own risk; review before use.

> **English summary** — A DeepSeek Harness skill for porting a character model from
> another game into *Mount & Blade II: Bannerlord* as an equippable outfit ("skin") mod:
> extracting the source mesh's **original skin weights**, mapping 400+ source bones onto
> Bannerlord's 27-bone humanoid rig (facial micro-bones → `head`, finger bones → `hand`,
> twist bones per limb segment), pose retargeting into the A-pose bind, building `.tpac`
> asset packages by template cloning (mesh / texture / material), writing the module +
> item XML, and a three-layer verification loop (pack integrity, shape metrics, in-game).
> Ships a symptom → root-cause table distilled from **three** full end-to-end ports and the
> iteration that followed in-game feedback, including the most expensive gotchas:
> **textures must declare `systemFlags=["has_alpha"]`** (or the engine treats them as opaque
> no matter what the material says); **helper bones matched by name can land on the wrong
> segment** (audit them by position); **source and target rigs disagree on pelvis/ankle
> conventions**; **the UV V-axis direction must be measured, not assumed**; and **asset GUIDs
> must be unique per package**, or two mods overwrite each other's textures when loaded together;
> **`.mgeo` filenames must be unique per material group**, or groups silently overwrite each other;
> and **weights must be quantised to u8 0–255** — writing the 0–1 floats straight through truncates
> them to 0/1 and the whole model turns into a rigid block that sways with the pelvis.

## 这是什么

DeepSeek Harness 的一个 **skill**：把外部角色模型做成《骑马与砍杀2：霸主》的装备（皮套）mod。
覆盖从源模型体检、原始蒙皮权重提取、骨骼映射与姿态重定向，到 `.tpac` 打包、装备 XML、
以及"移植后各类显示问题"的定位（颜色错位 / 纯黑或纯白片 / 脸型扭曲 / 手脚错位 / 站姿怪异）。

这份 skill 不是凭空写的 —— 它整理自**三次**完整的端到端移植工程
（RE:Resistance 的 Valerie Harmon、RE4 Remake 的艾达·王旗袍、仙剑奇侠传七 的月清疏/白茉晴 → 骑砍2 装备件），
所有数字、阈值、判据都是实测的，并且**记录了实机反馈后逐条定位修复的过程**。

> 第二版（RE4 旗袍）暴露出的、已回写进 skill 的关键教训：
> 辅助骨**按名字**映射会错段（`L_Toe_Twist_s` 名字是 Toe、位置在小腿中段 → 小腿不自然弯曲）；
> 源骨架与 BL 的**骨盆/脚踝约定差异**（髋部被撕开、鞋跟陷进地面）；
> **UV 的 V 轴方向必须实测**（第二次的源恰好与第一次相反）；
> 缝合片阈值不能照搬（照搬 6cm 删掉了 4.68% 的正常衣料）；
> 以及打包器固定随机种子导致**两个 mod 的资产 GUID 全部撞车**（只装一个正常、一起装就贴图紊乱）。

> 第三版（仙剑七，UE4/glTF 源、两名角色）新增的关键教训：
> **权重必须量化成 u8 0..255**（漏了这一步整块模型变刚体，而且那个函数"写对了但从没被调用"）；
> **`.mgeo` 文件名必须每组唯一**（同名覆盖 → "同一材质的袖子在、上衣不在"）；
> **验证必须落到最终 `.tpac`**，中间产物全绿没有意义；
> **离线摆姿势工装**（`pose_test.py`）—— 膝部错位/裙摆折痕这类问题只能在本地复现着调；
> **裙摆要左右对称跟腿**，按 `|x|` 分左右会在中缝剪出折痕；
> **判据里的绝对阈值只适配了一个模型**（长裙有效、短裙被整件削掉）；
> 以及 **XML 注释里不能有 `--`**（否则 mod 加载了但一件装备都没有）。

## 安装

skill 就是 `SKILL.md`（YAML frontmatter + 正文）。装到 DSH 的技能目录即可：

```bash
git clone https://github.com/tridkx/dsh-skill-bannerlord-character-mod \
  ~/.dsh/skills/bannerlord-character-mod
```

（Windows：`C:\Users\<你>\.dsh\skills\bannerlord-character-mod`）
DSH 在会话启动时扫描该目录；装好后 skill 名就是 `bannerlord-character-mod`。

## 里面有什么

`SKILL.md` 共 12 节，按"实际动手顺序"编排：

| 节 | 内容 |
|---|---|
| 0 | 交付物形态（一个装备 mod = SubModule.xml + items.xml + pack0.tpac 三个文件） |
| 1 | 源模型体检清单（决定难度与路线） |
| 2 | ★ 权重：优先用**原始蒙皮权重**，不要自算（OBJ 丢权重 ≠ 源文件没权重） |
| 3 | 骨骼映射 几百→27：面部微骨→head、手指→hand、扭曲骨按段；★ **名字必须用位置复核 + 位置校验否决 + 权重空间审计**；UE4 Biped 的三条额外规则（含**头发链强制归 head**） |
| 4 | 姿态重定向：Umeyama + 逐骨最小旋转 + 链式落点；手臂方案的量化判据；★ **骨架约定差异（骨盆/脚踝/颈与锁骨）、头部缩放渐隐、体型缩放的三条边界、长裙左右对称跟腿、缝合片阈值按模型重测**；★ **权重后处理（网格平滑 + 沿股骨宽混合带）** |
| 5 | `.tpac` 模板克隆打包；★ **UV 的 V 轴实测法**（UE4 与 RE 相反）；★ **权重必须量化成 u8 0..255**；★ **`.mgeo` 文件名必须每组唯一**；mip 链；**GUID 唯一性自查** |
| 6 | ★★ 贴图与透明：`has_alpha`、`modulate` vs `alphaTest`、★ **alpha 未必是镂空遮罩（先量直方图）**、`skinning` 布局、自生成贴图的 mip 陷阱 |
| 7 | 装备 XML 与槽位（头部槽只能穿一件；原版脚要靠 LegArmor 隐藏；`covers_*` 用差集定；★ **XML 注释里不能有 `--`**） |
| 8 | 五层验证（资产完整性 / **从最终 `.tpac` 倒推** / 形状残差 / ★★ **离线摆姿势看动画形变** / 实机） |
| 9 | 症状 → 根因速查表（30 条，含小腿异常弯曲 / 鞋跟陷地 / 长裙穿模 / **整块模型变刚体** / **同材质的部件一半不见** / **裙摆竖折痕** / **膝关节塌角** / 跨 mod 贴图紊乱） |
| 9.5 | Windows 编码与环境类坑（GBK 控制台、`rmtree` 参数、幂等性） |
| 10 | 方法论（参数差集、"像素数据 vs 资产元数据"、缓存键、先量化再动手、★ **产物身份值必须随工程变化**、★ **阈值必须按模型重测**、★★ **验证要落到最终产物且覆盖被调用路径**、★ **多角色要逐角色检查判据**） |
| 11 | 快速上手命令 + 自建工具清单 + **从 `.tpac` 读回顶点流的方法** |

## 配套仓库

| 仓库 | 内容 |
|---|---|
| [bannerlord-tpac-toolkit](https://github.com/tridkx/bannerlord-tpac-toolkit) | `.tpac` 读写/构建/校验工具集（C# CLI + Python 交叉验证 + 格式文档），本 skill 的所有 `mbtool` 命令来自它 |
| [bannerlord-xianjian7-outfits](https://github.com/tridkx/bannerlord-xianjian7-outfits) | 第三个完整工程（仙剑七 月清疏/白茉晴，UE4/glTF 源、双角色 6 件装备）：完整管线脚本 + 技术记录 17 节 + 验收渲染 |
| [bannerlord-ada-wong-costumes](https://github.com/tridkx/bannerlord-ada-wong-costumes) | 第二个完整工程（RE4R 艾达·王旗袍） |

> 本仓库与本 skill 都**不含任何游戏素材**（`.mesh/.tex/.tpac` 与提取结果均已排除）。

## 许可

MIT，见 [LICENSE](LICENSE)。
