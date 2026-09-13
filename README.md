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
> Ships a symptom → root-cause table distilled from a full end-to-end port, including the
> single most expensive gotcha: **textures must declare `systemFlags=["has_alpha"]`, or the
> engine treats them as opaque no matter what the material says.**

## 这是什么

DeepSeek Harness 的一个 **skill**：把外部角色模型做成《骑马与砍杀2：霸主》的装备（皮套）mod。
覆盖从源模型体检、原始蒙皮权重提取、骨骼映射与姿态重定向，到 `.tpac` 打包、装备 XML、
以及"移植后各类显示问题"的定位（颜色错位 / 纯黑或纯白片 / 脸型扭曲 / 手脚错位 / 站姿怪异）。

这份 skill 不是凭空写的 —— 它整理自一次完整的端到端移植工程（RE:Resistance 的 Valerie Harmon
→ 骑砍2 装备件），所有数字、阈值、判据都是实测的。

## 安装

skill 就是 `SKILL.md`（YAML frontmatter + 正文）。装到 DSH 的技能目录即可：

```bash
git clone https://github.com/tridkx/dsh-skill-bannerlord-character-mod \
  ~/.dsh/skills/bannerlord-character-mod
```

（Windows：`C:\Users\<你>\.dsh\skills\bannerlord-character-mod`）
DSH 在会话启动时扫描该目录；装好后 skill 名就是 `bannerlord-character-mod`。

## 里面有什么

`SKILL.md` 共 11 节，按"实际动手顺序"编排：

| 节 | 内容 |
|---|---|
| 0 | 交付物形态（一个装备 mod = SubModule.xml + items.xml + pack0.tpac 三个文件） |
| 1 | 源模型体检清单（决定难度与路线） |
| 2 | ★ 权重：优先用**原始蒙皮权重**，不要自算（OBJ 丢权重 ≠ 源文件没权重） |
| 3 | 骨骼映射 405→27：面部微骨→head、手指→hand、扭曲骨按段；交叉验证技巧 |
| 4 | 姿态重定向：Umeyama + 逐骨最小旋转 + 链式落点；手臂方案的量化判据；姿态微调旋钮 |
| 5 | `.tpac` 模板克隆打包；UV 的 V 轴；mip 链 |
| 6 | ★★ 贴图与透明：`has_alpha`、`modulate` vs `alphaTest`、`skinning` 布局、自生成贴图的 mip 陷阱 |
| 7 | 装备 XML 与槽位（头部槽只能穿一件；原版脚要靠 LegArmor 隐藏） |
| 8 | 三层验证（资产完整性 / 形状残差 / 实机） |
| 9 | 症状 → 根因速查表（15 条） |
| 10 | 方法论（做参数差集、"像素数据 vs 资产元数据"、缓存键、先量化再动手） |
| 11 | 快速上手命令 |

## 配套仓库

| 仓库 | 内容 |
|---|---|
| bannerlord-valerie-harmon-rebuild（已转私有，仅本地 `D:\dsh-mb-mod`） | 参考工程：完整的可复现管线脚本（权重提取/骨骼映射/重定向/形状体检/渲染验收/资产构建）+ PROGRESS.md 全部实测数字 |
| [bannerlord-tpac-toolkit](https://github.com/tridkx/bannerlord-tpac-toolkit) | `.tpac` 读写/构建/校验工具集（C# CLI + Python 交叉验证 + 格式文档），本 skill 的所有 `mbtool` 命令来自它 |

> 两个仓库与本 skill 都**不含任何游戏素材**（`.mesh/.tex/.tpac` 与提取结果均已排除）。

## 许可

MIT，见 [LICENSE](LICENSE)。
