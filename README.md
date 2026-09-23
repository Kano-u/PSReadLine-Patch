# PSReadLine Patch

给 [PowerShell/PSReadLine](https://github.com/PowerShell/PSReadLine) 打两个小补丁：

- `note-column.patch`：**把每条候选后面原本显示 `[来源]` 的位置改成显示 kc 的备注**，并**删掉选中项下方那块多余的灰色备注**（右侧已经实时显示了）
- `prediction-selection.patch`：新增只读的 `HasPredictionSelection()` API，让绑定 `↑` 的调用方能区分「预览里已有选中项」与「没有选中项」

本仓库只有补丁，不保存上游源码。两个补丁各自独立、互不重叠（前一个动 `Prediction.Entry.cs` / `Prediction.Views.cs`，后一个动 `Prediction.cs`），所以能干净地叠在同一份上游源码上。

## 效果

```
PS D:\> npx
   > npx create-react-app my-app                        创建 react 项目
   > npx tsc --noEmit                                   只做类型检查
   > npx @agegr/pi-web@latest                           pi web 网页端
   > npx view @agegr/pi-web version
   ⋮
   <kc(10)>
```

改造前，备注只能通过 `ToolTip` 显示，而 PSReadLine 只在候选被选中时才渲染 `ToolTip`：

```csharp
// PSReadLine/Prediction.Views.cs，改造前
if (_singleton._options.ShowToolTips && itemSelected && !string.IsNullOrWhiteSpace(entry.ToolTip))
```

改造后每行都能看到备注，所以选中项下方那块灰色 tooltip 就是重复信息了，一并删掉。

## 为什么改的是 ToolTip 而不是来源

每行候选只有两个可渲染的槽位：`SuggestionText`（插入命令行的原文，不能塞备注）和 `Source`。

`Source` 看着像是每行独立的，其实不是——它来自 `PredictionResult.Name`，**一次 `GetSuggestion` 调用只有一个值**，同批候选共用。这是 `ICommandPredictor` 的类型约定，插件侧改不了。

备注本来就在 `PredictiveSuggestion.ToolTip` 里逐条携带，PSReadLine 手上一直有它，只是渲染条件把它限制在选中项上。所以补丁改的是**那一行渲染条件**，把 `ToolTip` 放到每行的尾部槽位。

## 为什么需要 HasPredictionSelection

`↑`/`↓` 默认都被绑到 `PreviousHistory`/`NextHistory`，两者内部先调 `UpdateListSelection`：
当 list view 有选中项时，它们就在列表里移动；否则才去翻历史。

kc 把 `↑` 换成了「打开 kc TUI」的处理函数，因此丢掉了这个「已选中就在列表里走」的行为：
只要命令行不是多行，`↑` 就一定开 TUI，哪怕上一秒刚用 `↓` 在预览里选中了一行。

处理函数要自己恢复这个行为，就必须知道预览现在有没有选中项。
PSReadLine 只暴露了动作（`NextSuggestion`/`PreviousSuggestion`），没有暴露状态，所以补丁加一个只读判断：

```csharp
// PSReadLine/Prediction.cs
public static bool HasPredictionSelection()
{
    return _singleton._prediction.ActiveView is PredictionListView listView
        && listView.HasActiveSuggestion
        && listView.SelectedItemIndex >= 0;
}
```

它没有 `(ConsoleKeyInfo?, object)` 参数，所以不会出现在 `Get-PSReadLineKeyHandler` 的 unbound 列表里，也不需要登记到 `KeyBindings.cs` 的分组。

## 改动

| 文件 | 改动 |
| --- | --- |
| `PSReadLine/Prediction.Entry.cs` | 每行尾部不再渲染 `[SOURCE]`，改为渲染 `ToolTip`；备注用 `ListPredictionColor`（即原 `[kc]` 标签的橘色）；备注列左边缘对齐，宽度上限见下 |
| `PSReadLine/Prediction.Views.cs` | 新增常量 `NoteMaxWidth = 40`；删掉选中项下方灰色 tooltip 的渲染调用，并让 `HeightIsTooSmall` 不再为它预留高度 |
| `PSReadLine/Prediction.cs` | 新增 `public static bool HasPredictionSelection()` |

备注宽度 = `min(40, (列表宽 - 13) * 0.4)`，命令列在前、备注在后，两列各自固定，所以每一行的备注都从同一列开始。超长备注截断加 `…`。

`Source` 并没有消失：底部 `<kc(10)>` 那行和 `Ctrl+↑↓` 切换来源都仍然正常，它们自己维护来源列表，不读每行这个字段。

`F4`（在备用屏幕里看完整备注）也保留着，它走的是 `SelectedItemTooltip`，与删掉的灰色 tooltip 无关。

### 为什么不动那些变成死代码的成员

删掉渲染调用后，`RenderTooltip`、`GetToolTipLineCountForHeightCheck`、`_tooltipHeight`、`_maxTooltipHeight`、`TooltipMaxHeight` 就没有调用者了。它们**故意留着**：补丁要小，跟上游撞车的面积才小。清掉这几百行会让上游一改那块代码就打不上补丁，而留着只是几个不执行的成员，没有任何运行时影响。

`HeightIsTooSmall` 是例外，必须一起改——它仍在调 `GetToolTipLineCountForHeightCheck`，会为“已经不再渲染的 tooltip”预留 4 行。不改的话，选中带备注的候选时，终端稍矮整个列表会被误判“窗口太小”而消失。

## 用 kc 安装

补丁的安装与升级由 [kc](https://github.com/Kano-u/kc) 负责：

```powershell
kc psreadline-patch
```

它会问本机 pwsh 要 PSReadLine 版本和用户模块目录，然后克隆对应上游标签、应用补丁、本机编译、装进用户模块目录，最后新开一个 pwsh 验证是否真的加载到了补丁版。

需要本机有 `dotnet` SDK 和 `git`。装完要重启 PowerShell 窗口才生效。

## 手动安装

```powershell
# 1. 取版本号，假设是 2.4.5
Import-Module PSReadLine; (Get-Module PSReadLine).Version

# 2. 拉上游源码
git clone --depth 1 --branch v2.4.5 https://github.com/PowerShell/PSReadLine.git
cd PSReadLine

# 3. 打补丁（两个都要）
git apply /path/to/patches/note-column.patch
git apply /path/to/patches/prediction-selection.patch

# 4. 编译
dotnet publish -c Release -f netstandard2.0 PSReadLine/PSReadLine.csproj
dotnet build -c Release Polyfill/Polyfill.csproj

# 5. 按官方布局装配到用户模块目录
$dest = Join-Path ([Environment]::GetFolderPath("MyDocuments")) "PowerShell\Modules\PSReadLine\2.4.5"
New-Item "$dest\net6plus", "$dest\netstd" -ItemType Directory -Force
Copy-Item PSReadLine/bin/Release/netstandard2.0/publish/Microsoft.PowerShell.PSReadLine.dll $dest
Copy-Item PSReadLine/bin/Release/netstandard2.0/publish/Microsoft.PowerShell.Pager.dll $dest
Copy-Item PSReadLine/PSReadLine.psm1, PSReadLine/PSReadLine.psd1, PSReadLine/PSReadLine.format.ps1xml $dest
Copy-Item Polyfill/bin/Release/net6.0/Microsoft.PowerShell.PSReadLine.Polyfiller.dll "$dest\net6plus"
Copy-Item Polyfill/bin/Release/netstandard2.0/Microsoft.PowerShell.PSReadLine.Polyfiller.dll "$dest\netstd"
```

模块目录里少任何一个文件都会在导入时抛异常，尤其是 `Polyfiller`：缺了它 `PSConsoleReadLine` 的静态构造会直接失败。

## 注意事项

- **不向上游提交。** 这是个人自用改动，`[来源]` 换成备注是产品取向问题，不是上游的 bug。
- **升级 PowerShell 后要重跑。** PSReadLine 随 PowerShell 一起升级，版本号变了就得对着新版本重新打一次补丁。`kc psreadline-patch` 每次都问本机当前版本，直接重跑即可。
- **只影响当前用户。** 装的是用户模块目录，`PSModulePath` 里它排在系统目录之前；系统目录（`WindowsApps` 下）是受保护的，本来也改不动。
- **多个预测器同时挂载时看不出谁是谁。** 每行不再标注来源，来源只能从底部元数据行和 `Ctrl+↑↓` 看。
