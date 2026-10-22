Vivado 找不到 Zynq-7000 器件
# Vivado 找不到 Zynq-7000 器件：排查與修復流程

适用症状：Vivado 2026.1 搜索 `xc7z020` 无结果，或报 `Project 1-596 ... part ... is not available`；器件支持看似已安装，License Manager 也显示有效授权。

## 结论摘要

Vivado 2026.1 使用 tier-based 授权。应区分 **器件文件是否安装** 与 **当前 Vivado 选中的授权 tier 是否支持该器件**。本机的 Zynq-7000 文件和 Basic/Enterprise 授权都存在，但当前 tier 为 `ALVEO`，因此器件列表只有 Alveo，`get_parts xc7z020*` 为空。

本机通过 `get_vivado_tier` 确认当前 tier；把 `XILINXD_LICENSE_FILE` 指向只含本机有效 Basic 授权项的独立许可证文件后，查询变为 `BASIC`，`get_parts -quiet xc7z020*` 返回包含 `xc7z020clg400-2` 的器件列表。无需重装 Vivado。

AMD 2026.1 文档指出，Basic、Core、Pro 和 Enterprise/Gold 均支持全部 Zynq-7000 器件；Alveo 器件需要专用 Alveo tier。可参考 [UG973 器件支持表](https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Device-Availability-by-Subscription-Tier) 和 [UG835 `get_vivado_tier`](https://docs.amd.com/r/en-US/ug835-vivado-tcl-commands/get_vivado_tier)。

## 排查流程

1. **确认 Vivado 路径和版本**

   在 Tcl Console 运行：

   ```tcl
   version -short
   puts $::env(XILINX_VIVADO)
   ```

   确认正在使用预期的 Vivado 2026.1 安装。

2. **查询实际 tier 和器件列表**

   ```tcl
   get_vivado_tier
   get_parts -quiet xc7z020*
   ```

   如果 tier 为 `ALVEO` 且 Zynq 查询为空，先修正授权选择；不要依据安装器勾选状态或磁盘上是否存在器件文件来判断运行时 tier。

3. **交叉核对授权与安装**

   License Manager 中 `Loaded License Tiers` 只表示找到哪些授权，不代表当前会话使用哪个 tier。授权行上的 `Okay` 和 `Host IDs Match: Yes` 表示该授权可用，也不代表当前 tier 已切到它。若 `get_vivado_tier` 已是 `BASIC` 或 `ENTERPRISE` 而 Zynq 仍为空，再继续查安装器的 SoCs/Zynq-7000 组件与器件数据库。

## 本机修复步骤

本机的合并 `Xilinx.lic` 同时包含 Alveo、Enterprise 和 Basic tier；Vivado 启动时实际选中了 Alveo。解决方法是让 `XILINXD_LICENSE_FILE` 显式指向只包含有效 Basic 授权项的许可证文件。

1. 保留完整原许可证文件，不删除或手工改写它。
2. 使用 Vivado License Manager 从授权账户导出/加载 Basic tier 的许可证文件。若使用现有合并许可证制作 Basic-only 文件，只保留其中带 `License_Tier:BASIC` 的原始签名授权记录，不改动记录内容、签名、Host ID 或有效期。本机测试文件含 4 条 Basic Vivado 授权记录，放在：

   ```text
   %APPDATA%\XilinxLicense\Vivado_Basic.lic
   ```

3. 将当前 Windows 用户环境变量 `XILINXD_LICENSE_FILE` 指向该文件：

   ```powershell
   [Environment]::SetEnvironmentVariable(
       'XILINXD_LICENSE_FILE',
       "$env:APPDATA\XilinxLicense\Vivado_Basic.lic",
       'User'
   )
   ```

4. 关闭所有 Vivado 进程，再新启动 Vivado。新进程才会读取更新后的用户环境变量。
5. 在 Tcl Console 验收：

   ```tcl
   get_vivado_tier
   get_parts -quiet xc7z020*
   ```

   预期 tier 为 `BASIC`，器件列表包含 `xc7z020clg400-2`。本机用 Vivado 2026.1 batch 模式完成了这项验证。

如果未来需要在 Alveo 工程中使用 Alveo tier，可以把 `XILINXD_LICENSE_FILE` 切换为对应的 Alveo 许可证文件；切换后重新启动 Vivado，再查询 `get_vivado_tier`。不要假定 License Manager 展示的多个 tier 会自动合并成一个同时支持全部器件的 tier。

## 回滚与注意事项

- 本机修改前的许可证备份保存在各许可证旁的 `*.before-tier-order-20261022.bak` 文件中；原始 Alveo/Enterprise/Basic 合并许可证已恢复并保留。
- 回滚本机用户环境变量：

  ```powershell
  [Environment]::SetEnvironmentVariable('XILINXD_LICENSE_FILE', $null, 'User')
  ```

  这会恢复为使用 Vivado 的默认许可证搜索路径。本机此前该变量未设置。
- 不要用 `sort=` 手工改写 AMD 生成的许可证。本机曾试验此法，Vivado 随即报告无有效许可证；已恢复所有原许可证文件。
- 不要为了解决当前 tier 选择问题卸载或重装 Vivado；本机重装后问题仍存在，tier 修正后同一安装能列出 Zynq 器件。
- 不要在工单、聊天或截图中公开许可证全文、Host ID、MAC 地址、邮箱或签名字段。
