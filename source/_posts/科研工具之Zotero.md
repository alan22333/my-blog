---
title: 科研工具分享：Zotero
description: 'Zotero 是一款功能强大且完全开源的文献管理工具'
tags: ['工具分享']
toc: false
date: 2025-06-28 14:47:38
categories:
---

Zotero 是一款功能强大且完全开源的文献管理工具，凭借其高效、灵活、跨平台的特性，成为全球科研工作者的得力助手。它不仅能大幅简化文献收集与整理流程，还通过丰富的功能模块，为学术研究的各个环节提供便利。

### 核心功能介绍

1. **智能文献抓取**：Zotero 支持从网页、数据库、图书馆目录等多种渠道，一键抓取文献的元数据、摘要甚至全文。在浏览器中安装插件后，访问知网、Web of Science 等学术网站时，点击插件即可自动识别页面文献信息，将其快速添加到文献库，无需手动录入。
2. **多样化文献管理**：Zotero 可存储 PDF、Word、Excel、电子书等多种格式的文献资料，并通过文件夹、标签等方式进行分类整理。用户还能创建智能分组，依据文献的作者、关键词、日期等属性，自动筛选归类，让海量文献井井有条。
3. **高效批注与笔记**：它内置的 PDF 阅读器支持高亮、批注、添加书签等操作，方便用户在阅读文献时标记重点内容。同时，用户可添加独立的笔记，记录阅读心得、研究思路等，还能将笔记与文献建立关联，方便后续查阅。
4. **自动引用与参考文献生成**：Zotero 拥有强大的引用功能，集成了数千种学术引用样式，如 APA、MLA、Chicago 等。在撰写论文时，只需在 Word 或 LaTeX 中安装相应插件，就能轻松插入规范的文中引用，并自动生成符合要求的参考文献列表，大幅减少排版时间。
5. **多设备同步与协作**：借助 WebDAV、坚果云等云存储服务，Zotero 可实现文献库在多设备间的实时同步。用户还能创建共享群组，与团队成员共享文献资源、讨论研究进展，提升团队协作效率。
6. **丰富的插件扩展**：Zotero 的插件生态十分丰富，通过安装不同插件，可实现文献去重、自动翻译、数据可视化等功能拓展，满足用户多样化的科研需求。


### 使用教程
略
B站找个视频看看或者摸索一下就知道了，很简单

### 记一下上手后的配置，主要是一两个好用的插件：
- 登陆自己的账号（如果有的话），同步自己的文献库实现多设备办公
- 打开Zotero中文社区，找到插件商店（https://zotero-chinese.com/plugins/）
- 下载几个好用的插件：Translate for Zotero（阅读翻译工具，支持一键添加笔记）、Better Notes for Zotero（顾名思义，支持模版功能）、Awesome GPT（AI阅读辅助，虽然我用的不多，感觉一般般）、Ethereal Style（更美观的样式显示）
- 配置插件：
    - Translate for Zotero：主要就是获取密钥，教程[教程](https://zotero.yuque.com/staff-gkhviy/pdf-trans)
    - Better Notes for Zotero：去模版社区选择自己中意的模版即可，文末分享自用模版
    - Awesome GPT：用Github账号白嫖个API key，网址[白嫖](https://github.com/chatanywhere/GPT_API_free)

模版：
```
<html>

<h2 style="color: #B72415; text-align: center;">
    ${topItem.getField("titleTranslation") ? `${topItem.getField("titleTranslation")}` : ''}
</h2>
<hr>
<h2 style="color: #B72415; text-align: center;">
    ${topItem.getField("title")}
</h2>

<table style="border-collapse: collapse; width: 100%;">


<tr>
<td><b>期刊: </b>${topItem.getField('publicationTitle')}</b>

<!-- 分区 -->
    <tr><td>
        <b>分区: </b>
        <!-- In Zotero7, the tags of Ethereal Style plugin are referenced. Please install Ethereal Style in advance. -->
        ${{
        let space = " ㅤㅤ ㅤㅤ"
        return Array.prototype.map.call(
          Zotero.ZoteroStyle.api.renderCell(topItem, "publicationTags").childNodes,
          e => {
            e.innerText =  space + e.innerText + space;
            return e.outerHTML
          }
          ).join(space)
        }}$
      </td></tr>
	  
<td><b>作者:</b> ${topItem.getCreators().map((v)=>v.firstName+" "+v.lastName).join("; ")}</td>
</tr>

<tr>
    <td><b>论文发表日期: </b>${topItem.getField("date").replace(/^(\d+)\/(\d+)$/, "$2/$1")}</td>
</tr>

<tr>
<td><b>笔记创建日期: </b>${new Date().toLocaleString()}</td>
</tr>

 <!-- 原文链接 -->
    <tr><td>
        ${(() => {
          const attachments = Zotero.Items.get(topItem.getAttachments());
          if (attachments && attachments.length > 0) {
            return `<b>原文链接: </b><a href="zotero://open-pdf/0_${attachments[0].key}">${attachments[0].getFilename()}</a>`;
          } else {
            return `<b>原文链接: </b>`;
          }
        })()}
      </td></tr>



<tr>
    <td><b>摘要: </b>${topItem.getField('abstractTranslation') || topItem.getField('abstractNote')}</td>
</tr>

</table>

<h3 style="color:  #E65100; background-color:  #FFF8E1;">💡结论及创新点</h3>
<blockquote>本文解决了什么<u>新的科学问题</u>？<br>● <br>提出了什么<u>新的研究思路</u>？<br>● <br>应用了什么<u>新的研究工具</u>？<br>● </blockquote>

<h3 style="color:#2E7D32; background-color:    #F6FAEF;">📚前言及文献综述</h3>
<blockquote>本研究的<u>必要性、紧迫性、可行性</u>是什么？<br>● <br>作者是如何<u>论述</u>的？<br>● <br>引用了哪些该领域的<u>开创性文献</u>？<br>● </blockquote>

<h3 style="color: #006064; background-color:   #E8FBFB;">🔬可参考点</h3>
<blockquote>本文针对<u>哪些问题</u>应用了什么<u>新方法</u>？<br>● <br>以期获得什么<u>新效果</u>？<br>● </blockquote>

<h3 style="color:#4A69D3; background-color:   #EAF1FD;">📜获得思路</h3>
<blockquote>本文得出了什么<u>新的方法</u>？<br>● <br>通过本文在<u>写作上的收获</u>？<br>● </blockquote>

</html>
```