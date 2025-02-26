---
title: <% tp.file.title %>
date: <% tp.date.now("YYYY-MM-DD HH:mm:ss") %>
toc: true
categories:
<%*
// 自动生成分类（兼容Windows路径）
let folders = await tp.file.folder(true).split(/[\\/]/); 
folders = folders.slice(2); // 移除 "_posts" 父级目录
for (let category of folders) {
    tR += `  - ${category}\n`;
}
if (folders.length === 0) {
    tR += "  - 未分类";
}
%>
tags:
---

