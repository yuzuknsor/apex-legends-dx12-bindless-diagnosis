# Apex Legends DX12 Bindless 帧率异常

> [English](en/README.md) · [报告全文](REPORT.md) · [Full report (English)](en/REPORT.md)

Apex Legends 在 DX12 迁移后帧率断崖式下降的诊断记录，以及解决该问题的三个控制台变量。

## 修复方法

Steam → 库 → 右键 Apex Legends → 属性 → 启动选项，填入：

```
+staticprop_color_use_bindless 0 +staticprop_depth_use_bindless 0 +r_drawworld_depth_bindless 0
```

## 原理

Apex 的 DX12 渲染器对绘制调用数量最高的三类几何启用了 **bindless 资源绑定**：静态道具颜色通道、静态道具深度通道、世界几何深度通道。三者并集覆盖几乎全部不透明几何提交。

在该机器上，这条路径产生的是净负收益，且开销随可见图元数量变化。关闭它并非「提速」，而是移除了一项代价接近一个数量级的优化。

三个变量均为引擎内置控制台变量，不修改游戏文件。EA 官方支持渠道亦公开提供同一组参数作为 DX12 故障排查手段，并给出下一级开关 `+rhi.dx12.EnableGPUUploadHeaps 0`。

## 适用症状

- 大版本更新后帧数远低于预期，硬件健康，其他 CPU 密集型游戏正常
- 缺口随场景复杂度变化：对局中比靶场更差
- 重装驱动、重装游戏、重装系统均无效
- 无 TDR、无 WHEA、不崩溃——机器稳定，只是慢

若你的缺口是百分比级而非倍数级，本文大概率不适用。报告中说明了为何这个区分是关键的。

## 文件

| 路径 | 内容 |
|---|---|
| [`REPORT.md`](REPORT.md) | 诊断报告全文（中文） |
| [`en/REPORT.md`](en/REPORT.md) | Full diagnostic report (English) |
| [`DISCLAIMER.md`](DISCLAIMER.md) | 免责声明 |
| [`en/DISCLAIMER.md`](en/DISCLAIMER.md) | Disclaimer (English) |
| [`LICENSE`](LICENSE) | CC0 1.0 |

## 验证步骤

1. 改动前先记录基线（靶场与一局对战，各记 1% low）。
2. 加入参数，重启游戏，重新测量。
3. 保留基线数据。若后续补丁移除了非 bindless 回退路径，这三个变量会静默失效，且不报错。

## 许可

[CC0 1.0](LICENSE)，已释放至公有领域。可自由复制、修改、翻译、再分发、商用，无需许可，无需署名。

Apex Legends 及相关名称为 Electronic Arts Inc. 的商标。本仓库为非官方内容，与 EA 及 Respawn Entertainment 无隶属或授权关系。
