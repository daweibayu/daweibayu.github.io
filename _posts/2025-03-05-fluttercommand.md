---
layout: post
title:  "Flutter 命令行"
author: "daweibayu"
tags: Flutter
excerpt_separator: <!--more-->
---

没啥技术含量，gpt 生成，纯粹备忘

以下是基于搜索结果的Flutter常用命令整理表格，涵盖项目创建、依赖管理、构建运行、调试分析等核心场景：

| **命令**  | **功能描述**  | **参数示例/说明**  |
| ----------- | ----------- | ----------- |
| `flutter create` | ********** | ********** |
| `flutter run` | ********** | ********** |
| `flutter attach` | ********** | ********** |
| `flutter pub get` | ********** | ********** |
| `flutter pub upgrade` | ********** | ********** |
| `flutter analyze` | ********** | ********** |

| `flutter pub add ` | ********** | ********** |
| `flutter clean` | 删除 build/ 和 .dart_tool/ 目录 | 解决依赖下载异常 |
| `flutter pub cache repair` | flutter pub cache repair | 解决依赖下载异常 |
| `flutter test` | 运行单元测试 | 运行单元测试 |
| `flutter drive` | flutter drive | ********** |
| `flutter gen-l10n` | ********** | ********** |
| `flutter screenshot` | ********** | ********** |
| `flutter upgrade` | 升级Flutter SDK和依赖包 | --force 强制升级 |
| `flutter downgrade` | 回退到当前渠道的上一个稳定版本 | 适用于版本兼容性问题 |
| ********** | ********** | ********** |





| **项目创建**     | `flutter create <目录名>` | 创建新的Flutter项目                                                         | `--org`（包名）、`--platforms`（指定平台）、`-t`（模板类型，如app/plugin）             |   |
| **依赖管理**     | `flutter pub get`        | 安装`pubspec.yaml`中声明的依赖                                             | 等同于`flutter packages get`                                                       |   |
|                  | `flutter pub add <包名>`  | 添加新依赖包到项目                                                          | 例如`flutter pub add provider`                                                     |               |
|                  | `flutter pub upgrade`    | 升级所有依赖到最新版本                                                     | 可与`--major-versions`搭配强制主版本升级                                              |           |
| **构建与运行**   | `flutter run`            | 在连接的设备上运行应用                                                     | `-d`（指定设备）、`--release`（生产模式）、`--profile`（性能分析模式）                     |   |
|                  | `flutter build apk`      | 构建Android APK文件                                                        | `--split-per-abi`（生成多ABI包）、`--no-shrink`（禁用代码压缩）                          |           |
|                  | `flutter build ios`       | 构建iOS应用                                                                | 需Xcode环境支持                                                                     |               |
| **调试与分析**   | `flutter doctor`         | 检查开发环境配置完整性                                                     | `-v`显示详细诊断信息                                                               |   |
|                  | `flutter analyze`        | 静态分析Dart代码                                                           | 检测潜在代码问题                                                                     |       |
|                  | `flutter logs`           | 查看设备运行日志                                                           | `-c`清除历史日志                                                                     |               |
| **热重载/重启**  | `r`（控制台输入）         | 热重载（保留应用状态）                                                     | 需在`flutter run`运行时使用                                                          |           |
|                  | `R`（控制台输入）         | 热重启（重置应用状态）                                                     | 同上                                                                              |           |
| **设备管理**     | `flutter devices`        | 列出所有连接的设备                                                         | `--machine`输出设备详细信息                                                         |   |
|                  | `flutter emulators`      | 管理模拟器                                                                 | `--launch`启动模拟器、`--create`创建新模拟器                                           |           |
| **版本管理**     | `flutter upgrade`        | 升级Flutter SDK和依赖包                                                    | `--force`强制升级                                                                   |  |
|                  | `flutter downgrade`      | 回退到当前渠道的上一个稳定版本                                             | 适用于版本兼容性问题                                                                 |           |
| **清理与缓存**   | `flutter clean`          | 删除`build/`和`.dart_tool/`目录                                            | 常用于解决构建异常                                                                   |       |
|                  | `flutter pub cache repair` | 修复Pub包缓存问题                                                         | 解决依赖下载异常                                                                     |               |
| **测试**         | `flutter test`           | 运行单元测试                                                               | 支持指定测试文件或目录                                                               |       |
|                  | `flutter drive`          | 执行集成测试（Flutter Driver）                                             | 需编写Driver测试脚本                                                                 |           |
| **实用工具**     | `flutter screenshot`     | 截取设备屏幕截图                                                           | `--out=<路径>`指定保存位置                                                           |           |
|                  | `flutter gen-l10n`       | 生成国际化本地化文件                                                       | 需配置`l10n.yaml`                                                                   |               |

**说明**：
1. 部分命令支持组合参数，例如`flutter create --org com.example -i swift -a kotlin --platforms=android,ios`可定制跨平台项目；
2. 完整参数可通过`flutter <命令> --help`查看（如`flutter run --help`）；
3. 环境变量配置、IDE插件安装等辅助命令未列入表格，详见原始资料。