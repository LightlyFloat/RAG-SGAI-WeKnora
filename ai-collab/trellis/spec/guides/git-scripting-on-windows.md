# Windows 中文路径下的 Git 脚本规范

> 适用范围：在含中文文件名或中文目录名的仓库中编写任何 git 自动化脚本，
> 包括提交前的守卫检查、内容边界校验、批量文件筛选。
> 目的：避免因路径转义与编码差异导致脚本静默误判。

---

## 一、必须显式关闭路径转义

git 默认对非 ASCII 路径做八进制转义并加引号输出。`document/工作说明.md` 会输出为
`"document/\345\267\245\344\275\234\350\257\264\346\230\216.md"`。任何基于路径前缀
或路径关键字的判断都会因此失配。

**约定**：凡脚本要解析 git 输出的路径，一律加 `-c core.quotepath=false`。

```powershell
# 错误：中文路径被转义并带引号，前缀匹配失配，守卫误报
git diff --cached --name-only | Where-Object { $_ -notmatch '^document/' }

# 正确
git -c core.quotepath=false diff --cached --name-only | Where-Object { $_ -notmatch '^document/' }
```

受影响的子命令包括 `status --porcelain`、`diff --name-only`、`diff --name-status`、
`ls-tree --name-only`、`ls-files`。`git add` 与 `git rm` 接受未转义的真实路径作为
入参，不受此影响。

也可一次性配置 `git config core.quotepath false`，但脚本内显式传 `-c` 更可靠，
不依赖执行环境的既有配置。

## 二、守卫脚本必须先自证

内容边界类守卫的失败模式是**假阳性**：脚本因为自身缺陷而报警，掩盖了真实状态。
一次误报会中断流程并诱使人绕过守卫。

**约定**：守卫脚本先输出它实际取到的文件清单，再做判断，使误报可被立即识别。

```powershell
"=== 待提交 ==="; git -c core.quotepath=false diff --cached --name-only
$out = git -c core.quotepath=false diff --cached --name-only |
       Where-Object { $_ -notmatch '^(document/|delivery/)' }
if ($out) { "ABORT"; $out; exit 1 }
```

## 三、内容边界校验用文件数差值而非人工比对

同一仓库需向多个远端推送不同内容时，用可计算的等式表达边界，而非逐个目录检查。

```powershell
$d1 = (git ls-tree -r --name-only 交付分支 | Measure-Object -Line).Lines
$d2 = (git ls-tree -r --name-only 存档分支 | Measure-Object -Line).Lines
"差值应等于归档文件数：$($d2-$d1)"
git diff --name-only 交付分支..存档分支 | ForEach-Object { ($_ -split '/')[0] } | Sort-Object -Unique
```

第二条命令的输出应只含归档目录名。出现其他目录即说明边界已被破坏。

## 四、不要用 `.` 开头的目录承载需入库的内容

许多项目的 `.gitignore` 含 `.*` 规则以排除全部隐藏文件。若把需要入库的归档内容
放在 `.` 开头的目录下，`git add` 会静默跳过，且 `git status` 不提示。

**约定**：需入库的归档、记录、产物一律放在非 `.` 开头的目录中。确需入库被忽略的
路径时，用 `git add -f` 并在提交说明中写明原因。

## 五、检查清单

- [ ] 所有解析 git 路径输出的命令都加了 `-c core.quotepath=false`
- [ ] 守卫脚本在判断前先打印实际取到的清单
- [ ] 内容边界用文件数差值与目录集合两项同时校验
- [ ] 需入库的目录名不以 `.` 开头
- [ ] 推送后用 `git ls-remote` 复核远端 ref，不以本地成功为准
