---
title: 【PowerShell】将tab键的功能转换成更好用的自动补全
date: 2026-03-14 14:41:15
toc: true
categories:
  - 杂

tags:
  - 杂

---

打开配置文件
```ps
code $PROFILE
```

在配置文件中粘贴：
```ps
$smartTabHandler = {
    # 尝试执行右箭头的历史填充功能
    $line = $null
    $cursor = $null
    [Microsoft.PowerShell.PSConsoleReadLine]::GetBufferState([ref]$line, [ref]$cursor)
    [Microsoft.PowerShell.PSConsoleReadLine]::AcceptSuggestion()

    # 检查命令是否变化（历史填充是否生效）
    $newLine = $null
    $newCursor = $null
    [Microsoft.PowerShell.PSConsoleReadLine]::GetBufferState([ref]$newLine, [ref]$newCursor)
    
    # 如果命令未变化（没有历史建议），则执行 Tab 补全
    if ($line -eq $newLine) {
        [Microsoft.PowerShell.PSConsoleReadLine]::TabCompleteNext()
    }
}

# 设置智能 Tab 处理
Set-PSReadLineKeyHandler -Key Tab -ScriptBlock $smartTabHandler
```

重新加载
```ps
. $PROFILE
```

参考文章
[【PowerShell】将tab键的功能转换成更好用的自动补全_肖坤寄的技术博客_51CTO博客](https://blog.51cto.com/u_15423682/13957183)
