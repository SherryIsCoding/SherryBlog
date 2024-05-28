---
title: 浏览器扩展：URL-Manager
date: 2024-05-28 00:00:30
categories: 
    - [技术文档]
    - [工具]
tags: [脚本,工具]
---
最近在学习技术时每次打开浏览器都要打开固定的几个网站：Chatgpt，Github，CSDN，大佬的博客等等......
于是乎 想在Chrome设置 启动时打开特定网页或一组网页，设置后发现无效，
上网搜索发现也有人遇到了相似的问题，当然也给出了相应的解决方法
正好，最近想了解学习浏览器的扩展程序，就想着写一个脚本完成这个功能

感谢伟大的[谷歌文档](https://developer.chrome.com/docs/extensions/get-started?hl=zh-cn)，大概看了1个小时，跟着做了[Hello World扩展程序](https://developer.chrome.com/docs/extensions/get-started/tutorial/hello-world?hl=zh-cn#hello_world) 简单了解相关知识

### 大概思路
+ 调用[chrome.tabs.create()](https://developer.chrome.com/docs/extensions/reference/api/tabs?hl=zh-cn)使浏览器创建标签
+ 将url列表[存储](https://developer.chrome.com/docs/extensions/reference/api/storage?hl=zh-cn)到本地中
+ 使用[Service Worker](https://developer.chrome.com/docs/extensions/develop/concepts/service-workers/basics?hl=zh-cn)，监听浏览器启动并处理事件

### 概览
+ 允许用户添加、删除URL，并在浏览器启动时自动打开这些URL。
+ 防止重复的URL并限制URL的数量，以确保浏览器性能。

![图片](https://raw.githubusercontent.com/SherryIsCoding/SherryPic/img/img/url-manage.png)
[下载压缩包](https://github.com/SherryIsCoding/url-manager) 
下载压缩包后可直接加载到浏览器:
    进入[扩展程序页面](https://github.com/SherryIsCoding/url-manager)->打开开发者模式->点击加载已解压的扩展程序。

**好！动手开发**

### 步骤1：编写 manifest.json
manifest.json 文件描述了扩展程序的功能和配置。
+ "action"：定义扩展程序图标在 Google 工具栏中的外观和行为
+ "background"：指定包含扩展程序 Service Worker 的 JavaScript 文件，该 Service Worker 充当事件处理脚本。
+ "permissions"：允许使用特定的扩展程序 API。
更多关于清单文件知识，请查看官方文档[清单文件格式](https://developer.chrome.com/docs/extensions/reference/manifest?hl=zh-cn)

{% codeblock lang:json %}
{
  "manifest_version": 3,
  "name": "URL 管理器",
  "version": "1.0",
  "description": "管理和自动打开一组URL的扩展",
  "permissions": [
    "tabs",
    "storage"
  ],
  "background": {
    "service_worker": "background.js"
  },
  "action": {
    "default_popup": "popup.html"
  },
  "icons": {
    "16": "icon16.png",
    "48": "icon48.png",
    "128": "icon128.png"
  }
}
{% endcodeblock %}


### 步骤2：编写 background.js
background.js 文件注册为Service Worker，通过监听 [runtime.onStartup()](https://developer.chrome.com/docs/extensions/reference/api/runtime?hl=zh-cn#event-onStartup) 事件来自动打开存储的URL。
Service Worker 无法直接访问窗口对象，因此无法使用 window.localStorage 存储值。此外，Service Worker 也是短期执行环境；它们会在用户的浏览器会话中反复终止，这使得它们与全局变量不兼容。使用 [chrome.storage.local](https://developer.chrome.com/docs/extensions/reference/api/storage?hl=zh-cn#property-local)，将数据存储在本地机器上,只有在移除扩展程序时清除，存储空间上限为 10 MB。
{% codeblock lang:javascript %}
chrome.runtime.onStartup.addListener(() => {
    chrome.storage.local.get('urls', (data) => {
        const urls = data.urls || [];
        for (const url of urls) {
            chrome.tabs.create({ url });
        }
    });
});
{% endcodeblock %}

### 步骤3：创建弹出式窗口并设置样式
不是重点，请查看[源码](https://github.com/SherryIsCoding/url-manager)popup.html、popup.css

### 步骤4：编写 popup.js
popup.js 文件处理用户交互，包括添加和删除URL，以及防止重复和数量限制。

{% codeblock lang:javascript %}
const MAX_URLS = 10; // 设置URL数量限制

// 初始化时加载URL列表
document.addEventListener('DOMContentLoaded', loadUrls);

// 增加URL行
document.getElementById('add-url').addEventListener('click', () => addUrlRow(''));

function loadUrls() {
  chrome.storage.local.get('urls', (data) => {
    const urls = data.urls || [];
    for (const url of urls) {
      addUrlRow(url);
    }
  });
}

function addUrlRow(url = '') {
  const urlList = document.getElementById('url-list');
  const currentUrls = Array.from(urlList.querySelectorAll('input')).map(input => input.value.trim());

  // 检查是否超过最大数量限制
  if (currentUrls.length >= MAX_URLS) {
    alert(`你最多只能添加 ${MAX_URLS} 个URL。`);
    return;
  }

  // 创建新行
  const row = document.createElement('div');
  row.className = 'url-row';

  const input = document.createElement('input');
  input.type = 'text';
  input.value = url;
  row.appendChild(input);

  const removeButton = document.createElement('button');
  removeButton.textContent = '-';
  removeButton.addEventListener('click', () => {
    row.remove();
    saveUrls();
  });
  row.appendChild(removeButton);

  urlList.appendChild(row);
  input.addEventListener('change', () => {
    if (isDuplicateUrl(input.value, currentUrls)) {
      alert('这个URL已经存在。');
      input.value = '';
    } else {
      saveUrls();
    }
  });

  saveUrls();
}

function isDuplicateUrl(url, urlList) {
  return urlList.includes(url.trim());
}

function saveUrls() {
  const urlList = document.getElementById('url-list');
  const urls = [];
  urlList.querySelectorAll('input').forEach(input => {
    if (input.value.trim()) {
      urls.push(input.value.trim());
    }
  });
  chrome.storage.local.set({ urls });
}
{% endcodeblock %}