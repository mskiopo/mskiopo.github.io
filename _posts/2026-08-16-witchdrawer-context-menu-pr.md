---
title: 给桌面收纳工具加右键菜单，顺便救回了一个软件
date: 2026-08-16 00:50:00 +0800
categories: [技术]
tags: [WPF, Win32, C#, 右键菜单, 复盘]
---

> 给开源项目 WitchDrawer 加一个"收纳盒图标右键弹出原生菜单"的功能，从需求到发 PR 走了一遍。中间踩了 COM 互操作、Win11 提权、DPI 坐标几个坑，还顺带发现把一个软件拖坏了。写下来，留个复盘。

## 项目背景

[WitchDrawer](https://github.com/witchscottishfoldcat/WitchDrawer) 是一个基于原生 WPF 的 Windows 桌面文件收纳工具。把常用文件拖进桌面上的小收纳盒，快速打开，不占地方。技术栈是 .NET 10 + WPF + SQLite，刻意不碰 Electron。

要做的事，一句话：让收纳盒里的图标，右键能弹出和桌面、资源管理器一样的原生菜单。

## 一、手写 IContextMenu，第一个坑是 COM 签名

要让菜单和桌面一模一样，得调 Windows 的 `IContextMenu` 这个 COM 接口。这条路能拿到原生菜单，包括打开方式、属性、第三方右键扩展（7-Zip、Git 那类），代价是要手写互操作。

手写完编译，卡在一个很典型的错上：

```
error CS0199: 无法将静态只读字段用作 ref 或 out 值
```

原因很简单：`static readonly` 的 `Guid`（接口的 IID）不能直接当 `ref` 参数传。得先复制到局部变量，再传 `ref`。这个错本身不值一提，值一提的是：**COM 互操作这种事，编译过和能跑是两码事。**

真正能跑起来要过的关在后面。

## 二、真机冒烟测试暴露了第二个坑

编译过了，加了一个真机测试：对一个真实文件调 `CreateForPath`，验证 COM 链路能不能跑通。结果报：

```
GetUIObjectOf(IContextMenu) failed with HRESULT 0x80070057
```

`0x80070057` 是 `E_INVALIDARG`，参数错。

排查下来是 PIDL 的问题。`ILCreateFromPathW` 返回的是**绝对 PIDL**，而 `GetUIObjectOf` 要的是**相对于父文件夹的单个子 PIDL**（最后一层）。用 `ILFindLastID` 把最后一层提出来，就好了。

这个坑是靠"真机冒烟测试"抓出来的。如果只跑单元测试、只编译，根本不会发现，因为单测里没有真实 Shell。

## 三、Win11 上"以管理员身份运行"变成了要输入密码

菜单能弹出来了，但点"以管理员身份运行"，弹的不是 UAC 提权，而是"以其他用户身份运行"，让输入账号密码。

写了个诊断小程序，把菜单里每个项的 verb 打出来，发现 exe 的右键菜单里其实有**两个**相关的 verb：`runas`（以管理员身份运行）和 `runasuser`（以其他用户身份运行）。

Win11 的现代菜单在视觉上把它们合并成"以管理员身份运行"一个，但用命令 ID 偏移去调 `InvokeCommand` 时，会把提权退化成 `runasuser`，于是要密码。

修法是：`runas` 和 `runasuser` 统一改成走 `ShellExecuteEx(lpVerb="runas")`，强制触发 UAC 提权，不走会退化的偏移调用。

一度还想顺手把 `runasuser` 从菜单里过滤掉，结果"以管理员身份运行"整个消失了。因为 Win11 里这两个 verb 是绑在一组的，删一个连另一个一起没了。回退，只留提权分流。

## 四、菜单弹到"乱七八糟"的位置

能用了，但菜单弹出位置不对，不贴着图标，到处乱飘。

原因在坐标。一开始用 `e.GetPosition(null)` 拿鼠标位置，这在 WPF 里返回的是相对窗口客户区的坐标，不是屏幕物理坐标，再叠上 DPI 缩放和多显示器，`TrackPopupMenu` 拿到的坐标就是错的。

改成 P/Invoke `GetCursorPos` 拿物理屏幕坐标，位置就对了。

## 五、顺带发现自己把一个软件拖坏了

排查的时候才注意到，收纳盒里有个 `multisim.exe`，是 NI Multisim 14.0 的**主程序本体**，不是快捷方式。

它原本应该在 `D:\Circuit Design Suite 14.0\` 里。被当成文件拖进收纳盒后，安装目录缺了主程序，软件就报"找不到文件"。

最后把它从桌面（中间又被挪到了桌面）复制回安装目录，重新建了快捷方式，Multisim 就正常了。

这个事本身是一条独立的教训：**软件分两种，拖进"会搬文件"的收纳盒之前要分清。**

## 经验提炼

**COM 互操作，编译过不等于能用。** `IContextMenu` 这类手写签名，编译通过只是第一关。要有能跑在真实环境里的冒烟测试，哪怕只是一个"对真实文件调一次 CreateForPath 不抛异常"的断言。

**PIDL 不是路径字符串。** 绝对 PIDL 和相对子 PIDL 是两个东西，`ILFindLastID` 这一步省不得。

**提权走 `runas`，别让 shell 用偏移去猜。** Win11 会把提权退化成交凭据窗口，显式走 `ShellExecuteEx(runas)` 才稳。也别去过滤 `runasuser`，它们绑在一起。

**屏幕坐标用 `GetCursorPos`。** WPF 的 `GetPosition` 是客户区坐标，交给 Win32 弹窗会错位。

**安装版软件，别拖本体。** 会搬文件的收纳盒，只适合放独立文件或快捷方式。便携软件、装进 Program Files 的软件，本体被搬走就是残缺。映射盒、留原位的引用，才是对的放法。

下次再做这类 Native 功能，给每个系统调用都配一个真机冒烟测试。
