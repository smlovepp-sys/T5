# T5 — Text → Physics Scene (physics extractor & stage)

简体中文说明 / Chinese README

## 概述
这是一个将自然语言描述映射为物理参数、场景对象和关键帧的轻量化管道。它使用一个文本编码器（仓库中用 T5 encoder 作为 embedder）来推断材质/状态对物理参数（如刚度 / 摩擦 / 弹性 / 湿度 / 撕裂强度 / 张力 / 物体类型）的影响，并将这些物理信息组装为 Prop、Constraint 和 Keyframe，以生成可用于物理驱动场景或短视频演示的 SceneDescription。

主要用途：
- 从文本生成物体物理属性与场景描述
- 基于 verb/agent/patient 识别交互并生成约束（抓取、穿戴、施加力等）
- 为后续导出到物理引擎（Unity/PhysX/自定义）准备数据结构

## 主要组成
- physics_extractor — 负责加载材质数据库（默认 `material_full_complete.json`），执行词到物理参数的映射；支持基于数据库的直接查表或基于嵌入的投影估计。
- physics_stage — 顶层管道，包含：
  - T5 wrapper（load_t5_clip）用于构建 embedding 接口
  - EmbedUtils：嵌入缓存与相似度功能
  - PropAssembler：从 prompt 提取物体/修饰词并生成 Prop
  - InteractionAnalyzer：基于动词推断交互约束
  - KeyframeBuilder：构建关键帧事件
  - PhysicsStage：对外统一接口，调用上述组件生成 SceneDescription

## 快速开始（示例）
先安装依赖（示例使用 CPU 版本的 PyTorch）：

```bash
python -m venv .venv
source .venv/bin/activate    # Linux / macOS
# .venv\Scripts\activate    # Windows PowerShell
pip install --upgrade pip
# 安装 PyTorch（CPU 版示例）；如果有 GPU，请按官方说明安装对应版本
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu
pip install transformers numpy scipy
```

提示：如果你希望使用 GPU，请安装带 CUDA 支持的 torch 包；transformers 会下载 T5 权重。

运行示例（在仓库根目录）：

1) 把两份源文件改为标准 Python 模块名（推荐）

```bash
# 如果当前文件名没有 .py 扩展，执行重命名以便正常导入
git mv physics_extractor physics_extractor.py
git mv physics_stage physics_stage.py
```

2) 创建一个示例运行脚本 run_example.py：

```python
from physics_stage import load_t5_clip, PhysicsStage

clip, tokenizer = load_t5_clip("t5-small")
ps = PhysicsStage(clip)
scene = ps.generate_scene("a torn wet cotton shirt on the floor, person holding a cup", is_video=True)
print(scene)
```

3) 运行：

```bash
python run_example.py
```

## 重要注意事项
- 材质数据库：PhysicsExtractor 默认尝试加载 `material_full_complete.json`。仓库中目前未包含该文件——建议提供一个包含材质词条的 JSON（键为材质词，值为 dict，包含 category 与若干参数值），例如：

```json
{
  "cotton": {"category": "织物", "stiffness": 0.2, "friction": 0.5, "elasticity": 0.3},
  "steel": {"category": "金属", "stiffness": 1.0, "friction": 0.9, "elasticity": 0.1}
}
```

- 文件名：当前仓库顶层文件名为 `physics_extractor` 和 `physics_stage`（无 `.py`）。为了兼容 Python 的 import 机制，建议重命名为 `physics_extractor.py` 和 `physics_stage.py`，或在运行时通过路径/直接执行来加载。

- 替换 encoder：仓库实现将 T5 用作“clip-like” 嵌入器；你可以用 CLIP 或任意 sentence-transformers 模型替换，但需实现 `tokenize`、`encode_text`/`get_embedding` 与 `_get_embedding_weight` 等接口（参见 `physics_extractor._get_text_embedding` 与 `physics_stage.load_t5_clip`）。

## 示例：material_full_complete.json 样例行（建议）
将材质数据库放在仓库根目录或运行时可访问的位置，文件名保持默认或在 PhysicsExtractor 构造时传入路径。

## 仓库结构（简要）
```
physics_extractor(.py)   # 材质 DB 加载、词->物理映射、修饰词规则
physics_stage(.py)       # 顶层管道：embed、assembler、analyzer、builder、PhysicsStage
README.md                # 你现在阅读的文件
# material_full_complete.json (建议添加)
```

## API 快速参考
- PhysicsExtractor(db_file: str = "material_full_complete.json")
  - extract_from_text(prompt: str, clip) -> Dict[param->value]
  - get_physics_with_context(base_word, object_class, modifiers, explicit_material, clip) -> physics dict

- PhysicsStage(clip_model)
  - generate_scene(prompt: str, is_video: bool = True) -> SceneDescription dataclass

## 后续建议（可选的仓库改善）
- 添加 `requirements.txt`（或 pyproject/poetry）和 CI 测试（基本导入测试）
- 提供 `material_full_complete.json` 的示例或精简版本
- 把当前顶层脚本重命名为 `.py` 并加入示例脚本与运行说明
- 增加导出器（导出到 Unity/JSON schema）示例

## 许可证
仓库中未包含 LICENSE 文件。请根据需要添加合适的开源许可证（MIT/Apache-2.0/等）。

---

如果你同意，我可以：
- 把这个 README 自动提交到仓库（我可以现在创建 README.md 并发起 commit），
- 或先生成 `requirements.txt` 和 `run_example.py`，再提交一并入库。

你希望我现在把 README.md 添加到仓库吗？