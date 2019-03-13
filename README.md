# weapp-component-debugger

[![NPM version][npm-image]][npm-url] [![changelog][changelog-image]][changelog-url] [![license][license-image]][license-url]

[npm-image]: https://img.shields.io/npm/v/weapp-component-debugger.svg?style=flat-square
[npm-url]: https://npmjs.org/package/weapp-component-debugger
[license-image]: https://img.shields.io/github/license/ufologist/weapp-component-debugger.svg
[license-url]: https://github.com/ufologist/weapp-component-debugger/blob/master/LICENSE
[changelog-image]: https://img.shields.io/badge/CHANGE-LOG-blue.svg?style=flat-square
[changelog-url]: https://github.com/ufologist/weapp-component-debugger/blob/master/CHANGELOG.md

"小老弟"微信小程序调试助手(自定义组件), 简称: 小D(ebugger)

![weapp-component-debugger](https://user-images.githubusercontent.com/167221/54182603-ed863400-44dc-11e9-8ae8-4faeab3b8ac9.png)

## 功能

* 查看页面信息
  * AppID 和页面 URL
* 获取当前小程序帐号信息
* 获取设备信息
* 获取微信小程序登录凭证(code)
* [切换调试开关](https://developers.weixin.qq.com/miniprogram/dev/api/wx.setEnableDebug.html)
  * 可以在正式版的小程序中打开调试, 在微信小程序右下角会有一个 vConsole 按钮
  * 点击按钮, 可以看见前端打印的日志和其他功能
* 替换接口路径(Match API URL)
  * 类似 `Fiddler` 的 `AutoResponder` 或 `Charles` 的 `Rewrite` 功能
  * 例如将 `https://api.domain1.com/module` 替换为 `https://domain2.com`
  * 修改后发送的请求的 URL 如果匹配了规则, 则会替换成对应的 URL
* 扫一扫打开当前小程序的页面
  * 用于扫码打开未上线的页面, 因为使用微信直接扫生成的二维码只能跳转到上线的页面
* 获取小程序本地数据缓存
  * 支持编辑和删除缓存
* 跳转到小程序的页面
  * 支持添加参数

## 使用方法

PS: 目前适用于 [Min](https://github.com/meili/min-cli) 来构建的小程序项目接入, 对于原生小程序项目或者其他构建方式, 请手工 copy 代码 :(

* 安装组件

  ```
  npm install weapp-component-debugger --save
  ```

* 引用组件(在页面中引用自定义组件)

  ```json
  "usingComponents": {
      "weapp-component-debugger": "weapp-component-debugger"
  }
  ```

* 使用组件(在页面中使用自定义组件)

  ```html
  <weapp-component-debugger></weapp-component-debugger>
  ```

## 示例

```
<template>
<weapp-component-debugger bindshare="setShareContent"></weapp-component-debugger>
</template>

<script>
import {
    reloadCurrentPage
} from 'weapp-commons-util';

export default {
    config: {
        enablePullDownRefresh: true,
        usingComponents: {
            'weapp-component-debugger': 'weapp-component-debugger',
        }
    },
    data: {
        shareContent: {}
    },
    onPullDownRefresh: function() {
        reloadCurrentPage();
    },
    setShareContent: function(event) {
        this.setData({
            shareContent: event.detail
        });
    },
    onShareAppMessage: function() {
        return this.data.shareContent;
    }
};
</script>
<style lang="less"></style>
```

## 初心

问题: 后端如何查看小程序的页面并联调自己本机的接口?

分析:

* 由于是在手机上查看小程序, 传统的配置 host 的方式会有点麻烦
* 本地查看小程序也是可以的, 但需要每一个后端开发人员安装小程序开发者工具, 并且熟悉前端的小程序构建, 这需要后端接触到很多前端的技术, 有较大的门槛
* 为每一个后端开发人员生成一个他们特定 IP 的小程序开发版也是不现实的, 这样会需要反复与前端进行沟通, 时常打断前端开发的节奏

因此我们现在的解决方案是:

* 后端开发人员微信安装"小程序开发助手"(微信小程序)
* 在"小程序开发助手"中, 可以清楚地看到每个小程序的线上/体验/开发的版本, 这样就不用反复去通知后端开发人员版本更新了, 更不需要反复地去发用于预览小程序的二维码
  * **需要在小程序后台给对应的微信用户添加开发者权限才能看到对应的小程序**
* 由前端开发人员**发布出一个体验版本**, 专门用于后端联调. 因为体验版相当于一个稳定的提测版本, 与前端自己的开发版本是分离的, 不会影响到前端自己的开发
  * 推荐使用[微信小程序开发者工具命令行小秘书](https://github.com/ufologist/weappdevtools-cli)来发布
* 由前端提供一个隐藏的调试页面, 用于后端修改接口的根路径, 这样后端就可以与自己本机的接口做联调了
  * 例如点击版本号 10 次, 进入到隐藏的调试页面