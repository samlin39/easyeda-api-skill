# PCB 格式文档

本章节详细介绍了嘉立创EDA PCB文件的格式规范。PCB文件主要包含以下几个部分：

- [通用格式](/cn/format/pcb/common.md)：文档头、画布、层、物理层、偏好等。
- [分区格式](/cn/format/pcb/partition.md)：分区定义。
- [基础图元](/cn/format/pcb/primitive.md)：网络、图元配置、分组、丝印配置、关联、属性等。
- [焊盘与过孔](/cn/format/pcb/pad_via.md)：焊盘和过孔。
- [形状图元](/cn/format/pcb/shape.md)：直线、圆弧、多边形体系、折线、填充、区域、覆铜、图片、泪滴等。
- [二进制对象](/cn/format/pcb/obj.md)：内嵌对象。
- [3D 外壳](/cn/format/pcb/3d.md)：外壳、折痕、实体、螺丝柱。
- [文字体系](/cn/format/pcb/text.md)：文字。
- [属性](/cn/format/pcb/attr.md)：属性。
- [尺寸工具](/cn/format/pcb/dimension.md)：尺寸标注。
- [封装体系](/cn/format/pcb/component.md)：元件实例。
- [设计规则](/cn/format/pcb/rule.md)：规则模板、规则定义、规则选择器。
- [拼版](/cn/format/pcb/panel.md)：拼版参数。

## 操作配方 (Recipes)

源码层操作配方（当官方 API 不支持时的 workaround）：

- [单独覆盖某个器件的 3D 模型](./recipe-component-3d-model-override.md)：只改单个器件 3D 模型，不动 footprint/网络/位号。
- [机械孔边缘加镀锡层](./recipe-mechanical-hole-tinning.md)：扩大 NPTH 铜环 + 阻焊开窗，与 GND 隔断。
