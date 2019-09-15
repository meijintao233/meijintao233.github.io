---
title: "谷歌扩展开发小记"
date: 2019-09-15 19:40:45
tags:
  - 前端
  - 谷歌扩展
  - frontend
  - chrome extension
---

尝试开发谷歌浏览器扩展，medium 免费阅读器，目前主要是需要对请求进行拦截，修改，

[github 仓库]https://github.com/meijintao233/medium-reader-extension

将一些常用的 API 记录如下：

## manifest.json

主要是设置一些位置，展示信息，类似微信小程序的配置 app.json。

- Permissions: 设置一些需要获取权限的功能

- browser_action: 设置 icon、界面等，浏览器级的扩展，界面 html 和页面的 html 是两个隔离的环境

- page_action: 设置 icon、界面等，页面级的扩展，界面 html 和页面的 html 是两个隔离的环境

- background: 设置 js 脚本等
  <!-- more -->

## content_script

前台脚本，不可跨域，会注入到页面中，可以操作当前页面的 dom 等

## background_script

后台脚本，可跨域，一般会需要从 content_script 中发送信息过来，再发跨域请求

## page_action

页面级的扩展，可控制显隐，特定页面显示

可绑定 onClick 事件，在点击扩展图标时执行，但是如果有 html 界面则该事件不生效

## browser_action

浏览器级扩展

可绑定 onClick 事件，在点击扩展图标时执行，但是如果有 html 界面则该事件不生效

## chrome api

- tabs

  get 获取标签页详情

  sendMessage 方法 向指定标签页中的 content_script 发送消息

  excuteScript 可以注入 js 脚本 参数是 (tabid,{file:[]})

  connect 用于所属扩展与 content_scirpt 的连接进行通信

  create 创建新标签页

  duplicate 复制标签页

  query 获取标签页

  highlight 标签页高亮

  update 更新标签页属性

  move 移动标签页 根据 index 从 0 开始计数

  remove 关闭标签页

* storage

  sync 对浏览器同步功能生效

  local 对本机生效

  get

  set

  clear

  remove

  getBytesInUse 获取使用空间

  onChange 发生变化触发

- runtime 相当于运行时功能

  getBackgroundPage 获取该扩展的 background 界面的 window 对象，并将对象以回调形式传入

  getManifest 获取 manifest.json 信息

  getUrl 将相对地址转换成绝对地址

  reload 重新加载扩展

  connect 用于 content_scirpt 连接所属扩展进行通信

  getPlatformInfo 获取平台信息

  sendMessage 方法 extensionId 参数 向其他扩展发送信息

  onInstalled 扩展安装成功或更新触发

  onSuspend 页面卸载前触发

  onConnect 从其他扩展或者 content_script 发起连接时触发

  onMessage 从其他扩展或者 content_script 接收消息时触发

* extension

  getViews 获取该扩展界面下的所有 window 对象，包括 background 界面

  getBackgroundPage 获取该扩展的 background 界面的 window 对象

  getUrl 将相对地址转换成绝对地址

## 通信、请求拦截

- Web request

有 onBeforeSendHeaders，onHeadersReceived 支持修改 header 信息，onCompleted 表示请求完成等事件

接受三个参数，分别是回调（回调中能获取请求的信息），filter 数组（type，urls (使用匹配模式)，），额外页面信息（blocking,extraheaders,requestheader,responseheaders 等），blocking 模式才能对请求进行修改，需要 return blocking 对象

[更多 API](https://developer.chrome.com/extensions/api_index)
