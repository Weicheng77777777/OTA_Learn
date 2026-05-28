# OTA_Learn
记录学习OTA过程

# GEC6818 第一次本地 OTA 测试记录

## 1. 测试目标

本次测试目标是验证 GEC6818 在 SD 卡启动环境下是否可以完成最简单的本地 OTA 流程。

本阶段 OTA 不更新 U-Boot，不更新 Kernel，不更新 DTB，也不更新完整 rootfs，只做最小闭环验证：

1. 制作本地 OTA 包
2. 拷贝 OTA 包到开发板
3. 校验 OTA 包
4. 覆盖少量 rootfs 文件
5. 更新版本号
6. 重启后确认系统仍可正常启动
7. 记录 OTA 后出现的问题

本次 OTA 从：

```text
v0.5.0-gpio-led-key-pwm
