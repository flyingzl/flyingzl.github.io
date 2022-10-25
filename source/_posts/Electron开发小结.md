---
title: Electron开发小结
date: 2020-09-11 13:00:00
categories:
- 前端技术
tags: 
- JavaScript
- Node.js
- Electron
---

最近基于Electron开发了一个音视频应用，遇到了一些坑，特此记录下，希望可以帮助后续的同学。

## Electron下载慢

安装`electron`时会自动最新的Electron二进制文件，由于文件比较大还容易墙，所以我们可以先配置好环境变量，再运行`yarn`或者`npm install`

```bash
export ELECTRON_MIRROR="https://npm.taobao.org/mirrors/electron/"
yarn install
```

`ELECTRON_MIRROR`表示`Electron`镜像地址，切换到国内淘宝源，速度会飞快 


## Electron 截图

Electron里面其实有专门的截屏函数[capturePage](https://www.electronjs.org/docs/api/web-contents#contentscapturepagerect), 但必须在主进程进行调用，可以在渲染进程通过[remote](https://www.electronjs.org/docs/api/remote)模快来调用，如下所示：

```javascript
const takeScreenshot = ( savedFolderName = 'screenShoots' ) => {
    const isNative = typeof require === 'function' && !!require('electron')
    return new Promise((resolve, reject) => {
        if(isNative) {
            const fs = require('fs')
            const path = require('path')
            const { remote } = require('electron')
            const screenshotsDir = path.join(process.env.HOME || process.env.USERPROFILE, savedFolderName)
            if (!fs.existsSync(screenshotsDir)) {
                fs.mkdirSync(screenshotsDir)
            }
            remote.getCurrentWindow().capturePage().then(img => {
                const name = Date.now()
                fs.writeFile(`${screenshotsDir}${path.sep}${name}.png`, img.toPNG(), err => {
                    if (err != null) {
                        reject(err)
                    } else {
                        resolve(screenshotsDir)
                    }
                })
            })

        } else {
            reject('should be invoked in electron environment')
        }
    })
}
```


## Electron 桌面分享

大家知道，在Google Chrome中可以调用[MediaDevices.getDisplayMedia()](https://developer.mozilla.org/zh-CN/docs/Web/API/MediaDevices/getDisplayMedia)来进行分享，不过很可惜， `Electron`没法使用该方法，具体issues可以参考 [getDisplayMedia with Chrome 72 throwing Not Allowed](https://github.com/electron/electron/issues/16513)。不过`Electron`中提供了[desktopCapturer](https://www.electronjs.org/docs/api/desktop-capturer)来获取桌面视频流，这样也可以进行屏幕共享。

为了保证代码兼容性，并且在`Electron`进行屏幕分享，我们可以重写`window.navigator.mediaDevices`方法，保证代码无缝迁移。代码放在`preload.js`中即可。

```javascript
const { desktopCapturer } = require('electron')
window.navigator.mediaDevices = window.navigator.mediaDevices || {}
window.navigator.mediaDevices.getDisplayMedia = async () => {
    // 获取所有可以分享的桌面或者窗口
    const sources = await desktopCapturer.getSources({ types: ['screen', 'window'] })
    // 为了方便，我们只选择全屏进行分享
    const source = sources.filter(source => source.name === 'Entire Screen' || source.name === 'Electron')[0]
    if (source) {
        try {
            const stream = await navigator.mediaDevices.getUserMedia({
                audio: false,
                video: {
                  mandatory: {
                    chromeMediaSource: 'desktop',
                    chromeMediaSourceId: source.id,
                    minWidth: 1280,
                    maxWidth: 1280,
                    minHeight: 720,
                    maxHeight: 720
                  }
                }
            })
            return stream
        } catch (e) {
            console.error(e)
        }
        return
    }
}

```

上面的例子只是进行全屏分享，如果想基于某个单独窗口分享，可以遍历`desktopCapturer.getSources({ types: ['screen', 'window'] })`返回的数据，弹出窗口，用户选择后进行分享。


## Electron 全屏

在网页中，我们可以通过[requestFullscreen](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/requestFullScreen)进入全屏，通过[Document.exitFullscreen](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/exitFullscreen)来退出全屏，不过在Electron，如果应用点击最大化按钮进入全屏后，上面的方法就失效了，我们可以在渲染进程中调用Electron的[setFullScreen](https://www.electronjs.org/docs/api/browser-window#winsetfullscreenflag)方法来控制全屏进入和退出 ，如下代码所示：

```javascript
const utils = {
    isNative() {
        return typeof require === 'function' && !!require('electron')
    }
}

const toggleFullScreen = () => {
    if (utils.isNative()) {
        const win = require('electron').remote.getCurrentWindow()
        // 检测当前Electron是否为全屏状态
        const isFullScreen =  win.fullScreen
        win.setFullScreen(!isFullScreen)
    }  else {
        const isFullScreen =  document.webkitIsFullScreen
        if (isFullScreen) {
            document.webkitExitFullscreen()
        } else {
            document.body.requestFullscreen()
        }
    }
}
```

## 关闭应用

这个比较简单，直接调用Electron中的[close](https://www.electronjs.org/docs/api/browser-window#winclose)方法即可。

```javascript
const close = () => {
    utils.isNative() && require('electron').remote.getCurrentWindow().close()
}
```

## electron-builder 打包后，input输入框粘贴、剪切失效的问题

这个问题很奇怪，直接通过`electron src/main/index.js` 运行时不会存在该问题，但是通过 [electron-builder](https://www.electron.build/) 构建后会存在该问题。我的Mac上就存在该问题。

解决方案有两个，第一个方案是给应用增加菜单，比如网上的解决方案：

[https://github.com/onmyway133/blog/issues/67](https://github.com/onmyway133/blog/issues/67)

```javascript
const {app} = require('electron')
const Menu = require('electron').Menu

app.on('ready', () => {
  createWindow()
  createMenu()
})

function createMenu() {
  const application = {
    label: "Application",
    submenu: [
      {
        label: "About Application",
        selector: "orderFrontStandardAboutPanel:"
      },
      {
        type: "separator"
      },
      {
        label: "Quit",
        accelerator: "Command+Q",
        click: () => {
          app.quit()
        }
      }
    ]
  }

  const edit = {
    label: "Edit",
    submenu: [
      {
        label: "Undo",
        accelerator: "CmdOrCtrl+Z",
        selector: "undo:"
      },
      {
        label: "Redo",
        accelerator: "Shift+CmdOrCtrl+Z",
        selector: "redo:"
      },
      {
        type: "separator"
      },
      {
        label: "Cut",
        accelerator: "CmdOrCtrl+X",
        selector: "cut:"
      },
      {
        label: "Copy",
        accelerator: "CmdOrCtrl+C",
        selector: "copy:"
      },
      {
        label: "Paste",
        accelerator: "CmdOrCtrl+V",
        selector: "paste:"
      },
      {
        label: "Select All",
        accelerator: "CmdOrCtrl+A",
        selector: "selectAll:"
      }
    ]
  }

  const template = [
    application,
    edit
  ]

  Menu.setApplicationMenu(Menu.buildFromTemplate(template))
}
```

因为我们Electron不需要菜单, 所以第一种方式不合适； 所以我换了一种方式，直接在`document`中监听`onkeydown`事件来手动处理，如下所示：

```javascript

const fixInputEvent = () => {
    const { clipboard } = require('electron')
    const keyCodes = {
        V: 86,
        C: 67,
        X: 88
    }
    document.onkeydown = function(event){
        let activeElement =  event.target
        if (activeElement.tagName  !== 'INPUT') return 

        const startOffset = activeElement.selectionStart
        const endOffset = activeElement.selectionEnd
        const clipboardText = clipboard.readText()

        if(event.ctrlKey || event.metaKey){  // detect ctrl or cmd
            switch(event.which) {
                case keyCodes.V: {
                    activeElement.setRangeText(clipboardText, startOffset, endOffset, 'end')
                    activeElement.dispatchEvent(new Event('input', { bubbles: true}))
                    break
                }
                case keyCodes.C: {
                    const text = activeElement.value.substring(startOffset, endOffset)
                    clipboard.writeText(text)
                    break
                }
                case keyCodes.X:  {
                    const text = activeElement.value.substring(startOffset, endOffset)
                    clipboard.writeText(text)
                    activeElement.setRangeText('', startOffset, endOffset )
                    break
                }
            }
        }
    }
}

```

上面的代码处理了 `CTRL + C`、`CTRL + V`已经`CTRL + X` 三种情况，也就是复制、粘贴和剪切三种场景， 用到了[HTMLInputElement.setRangeText()
](https://developer.mozilla.org/en-US/docs/Web/API/HTMLInputElement/setRangeText) 这个函数，解决了在指定光标处复制、粘贴以及剪切等问题。大家感兴趣可以看看。


## electron-webpack 构建问题

开发用到了[electron-webpack](https://github.com/electron-userland/electron-webpack) 构建工具，由于我们渲染进程的代码（前端HTML、JavaScript等代码）是其它项目已经写好的，所以不需要进行渲染进程打包，可以在`package.json`中关闭渲染进程打包。

```json
{
  "electronWebpack": {
    // 渲染进程无需打包
    "renderer": null
  }
}
```

既然关闭了渲染进程打包，那么我们已经写好的前端代码放在哪里呢？需要放在`static`文件夹， 并且在主进程中可以这样访问：

```javascript
const mainWindow = new BrowserWindow({
    width: 1280,
    height: 840,
    center: true,
    autoHideMenuBar: true,
    webPreferences: {
        webSecurity: false,
        preload: path.join(__dirname, 'preload.js'),
        nodeIntegration: true,
        enableRemoteModule: true,
    },
    title: 'XXX视频会议'
})

mainWindow.loadFile(path.join(__static, 'index.html'))
```

通过`electron-webpack`打包后，默认引入的第三方库都会默认设置为`external`, 所以如果引入了第三方库， 需要改下`package.json`, 增加`whiteListedModules`配置项，这样webpack才会把第三方法代码一起打包压缩。

```json
  "electronWebpack": {
    "renderer": null,
    "whiteListedModules": [
      "open"
    ]
  }
```

`electron-webpack`打包会默认生成`SourceMap`文件， 我们一般生产环境不需要`SourceMap`，程序运行时还会报`Uncaught Exception: Error: Cannot find module source-map-support/source-map-support.js` 错误， 原因是`electron-webpack`使用了[BannerPlugin](https://www.webpackjs.com/plugins/banner-plugin/)插件，默认会在打包后的文件中加入一句`require("source-map-support/source-map-support.js").install()`。

我们可以修改`electron-webpack`的配置文件来删除`BannerPlugin`插件， 在`package.json`中增加如下配置：

```json
  "electronWebpack": {
    "main": {
      "extraEntries": [
        "@/preload.js"
      ],
      "webpackConfig": "electron.config.js"
    },
    "renderer": null,
    "whiteListedModules": [
      "open"
    ]
}
```
`electron.config.js`内容如下所示：

```javascript

module.exports = function (config) {
    config.devtool = false
    config.plugins = config.plugins.filter(plugin => plugin.constructor.name !== 'BannerPlugin')
    return config
}
```

这样就可以解决`SourceMap`问题了。


## MAC及Windows打包

打包主要使用 [electron-builder](https://www.electron.build/)来进行打包，我的`package.json`配置如下：

```json


{
  "scripts": {
    "dev": "cross-env NODE_ENV=development electron src/main/index.js",
    "compile": "cross-env NODE_ENV=production electron-webpack",
    "mac": "rm -rf release dist && yarn compile && electron-builder --mac --x64",
    "windows": "rm -rf release dist && yarn compile && electron-builder --win --x64"
  },

  "build": {
    "appId": "larry.asyncoder.com",
    "mac": {
      "category": "asyncoder-web",
      "target": "dmg",
      "icon": "./icon.icns"
    },
    "win": {
      "target": "portable",
      "icon": "./icon.png"
    },
    "directories": {
      "output": "release/${platform}"
    }
  }
}
```

运行`yarn mac`和 `yarn windows`就可以生成Mac上的`dmg`和windows上的`exe`文件。

值得说明的是， 在Mac上是没法直接对Windows环境进行打包的， 所以推荐使用[docker](https://www.docker.com/)来进行打包。安装好`Docker`, 命令行进入工程目录，运行如下代码

```bash
docker run --rm -ti \
 --env-file <(env | grep -iE 'DEBUG|NODE_|ELECTRON_|YARN_|NPM_|CI|CIRCLE|TRAVIS_TAG|TRAVIS|TRAVIS_REPO_|TRAVIS_BUILD_|TRAVIS_BRANCH|TRAVIS_PULL_REQUEST_|APPVEYOR_|CSC_|GH_|GITHUB_|BT_|AWS_|STRIP|BUILD_') \
 --env ELECTRON_CACHE="/root/.cache/electron" \
 --env ELECTRON_BUILDER_CACHE="/root/.cache/electron-builder" \
 -v ${PWD}:/project \
 -v ${PWD##*/}-node-modules:/project/node_modules \
 -v ~/.cache/electron:/root/.cache/electron \
 -v ~/.cache/electron-builder:/root/.cache/electron-builder \
 electronuserland/builder:wine
```

进入以后，先运行`yarn`, 再运行`yarn windows`即可。


