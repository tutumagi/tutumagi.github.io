---
title: Code-Push使用经验(仅iOS)
date: 2016-05-11 03:28:42
tags:
- Code-Push
category:
- Coding 
---

## 安装react-native-code-push
参考[React-native-code-push README](https://github.com/Microsoft/react-native-code-push#getting-started)  

1. 安装 `npm install react-native-code-push --save`
    参数 --save 在安装的同时，把信息写入package.json 文件的项目路径  
    检查:  
    ① package.json文件加入了
    
    ```json
     {
        "name": "APPReact",
        "version": "0.0.1",
        "private": true,
        "scripts": {
          "start": "node node_modules/react-native/local-cli/cli.js start"
        },
        "dependencies": {
          "react": "^0.14.8",
          "react-native": "^0.25.1",
          "react-native-code-push": "^1.10.6-beta"
        }
      }
    ```
    ② `node_modules`下有`react-native-code-push`文件夹
2. 安装到本地工程  
   主要是将react-native-code-push工程加入到我们的本地工程里面去，[参考官方文档。](https://github.com/Microsoft/react-native-code-push#getting-started)
    **iOS**中修改加载js文件的路径代码，推荐写法：
    
    ```objective-c
    NSURL *jsCodeLocation;

    #ifdef DEBUG
        jsCodeLocation = [NSURL URLWithString:@"http://localhost:8081/index.ios.bundle?platform=ios&dev=true"];
    #else
        jsCodeLocation = [CodePush bundleURL];  //这里默认是main.jsbundle的js文件名，如果是其他文件名，请使用其他接口来获取bundleURL
    #endif
    ```

## 安装&配置CodePush
参考[CodePush Management](https://github.com/Microsoft/code-push/blob/master/cli/README.md)

1. 安装  
`npm install -g code-push-cli`  

    ```
    //指令列表
    tufei$ code-push
      _____        __    ___           __ 
     / ___/__  ___/ /__ / _ \__ _____ / / 
    / /__/ _ \/ _  / -_) ___/ // (_-</ _ \
    \___/\___/\_,_/\__/_/   \_,_/___/_//_/    CLI v1.10.0-beta
    ======================================
        
    CodePush is a service that enables you to deploy mobile app updates directly to your users' devices.
        
    Usage: code-push <command>
        
    命令：
      access-key       View and manage the access keys associated with your account
      app              View and manage your CodePush apps
      collaborator     View and manage app collaborators
      deployment       View and manage your app deployments
      link             Link an additional authentication provider (e.g. GitHub) to an existing CodePush account
      login            Authenticate with the CodePush server in order to begin managing your apps
      logout           Log out of the current session
      patch            Update the metadata for an existing release
      promote          Promote the latest release from one app deployment to another
      register         Register a new CodePush account
      release          Release an update to an app deployment
      release-cordova  Release a Cordova update to an app deployment
      release-react    Release a React Native update to an app deployment
      rollback         Rollback the latest release for an app deployment
      whoami           Display the account info for the current login session
    ```
2. 配置  
    ① 注册code-push账号  
      `code-push register`弹出窗口，使用Microsoft账号或者GitHub账号，会得到token然后拷贝到命令行提示你输入的地方，这时候默认你已经注册好Code-Push账号并且已经登陆了，可以使用`code-push login / code-push logout`登入，登出  
        * 使用code-push 服务器  
        * 登陆 code-push login  
        * 注销 code-push logout  
        * 列出 登陆的token      
        * code-push access-key ls  
        * 删除某个access-key  
        * code-push access-key rm <accessKey>  
        
    ② 管理app

        ```
        tutumagideMBP:APPReact tufei$ code-push app
        Usage: code-push app <command>
        
        命令：
          add       Add a new app to your account
          remove    Remove an app from your account
          rm        Remove an app from your account
          rename    Rename an existing app
          list      Lists the apps associated with your account
          ls        Lists the apps associated with your account
          transfer  Transfer the ownership of an app to another account
        ```
    先把我们的APP加入进去，`Deployment key`是我们在Native端需要配置的，也是我们热更新时将不同环境（开发环境，测试环境，生产环境）隔离开来的**`key`**，默认生成两个环境，我们可以使用`code-push deployment add APP_task test`添加一个环境
    
        ```
        $code-push app add APP_task
        Successfully added the "APP_task" app, along with the following default deployments:
        ┌────────────┬───────────────────────────────────────┐
        │ Name       │ Deployment Key                        │
        ├────────────┼───────────────────────────────────────┤
        │ Production │ CKUhCYrVOt9puAYYOEyaBGyQ9gFHEk2EyF5-- │
        ├────────────┼───────────────────────────────────────┤
        │ Staging    │ 2WuVTI0Ahb9mHoNkOufG9eBCGQxsEk2EyF5-- │
        └────────────┴───────────────────────────────────────┘
        ```
    查看我们添加的APP

        ```
        tutumagideMBP:APPReact tufei$ code-push app ls
        ┌───────────┬─────────────────────┐
        │ Name      │ Deployments         │
        ├───────────┼─────────────────────┤
        │ APP_task │ Production, Staging │
        └───────────┴─────────────────────┘
        ```
    添加一个测试环境

        ```
        $ code-push deployment add APP_task test
        Successfully added the "test" deployment with key "2ZKF6OuLvf0BPhOKfs0b3IiSHjFJEk2EyF5--" to the "APP_task" app.
        ```
    查看APP的环境信息

        ```
        tufei$ code-push deployment list APP_task --format json
        [
          {
            "name": "dev",
            "key": "0MbMLGx_gPn7uO_HxWrW9YhFjFMBEk2EyF5--",
            "package": null
          },
          {
            "name": "Production",
            "key": "CKUhCYrVOt9puAYYOEyaBGyQ9gFHEk2EyF5--",
            "package": null
          },
          {
            "name": "Staging",
            "key": "2WuVTI0Ahb9mHoNkOufG9eBCGQxsEk2EyF5--",
            "package": null
          },
          {
            "name": "test",
            "key": "2ZKF6OuLvf0BPhOKfs0b3IiSHjFJEk2EyF5--",
            "package": null
          }
        ]
        ```
    将我们需要运作的环境的key添加到XCode的Target Info.plist里面，添加`CodePushDeploymentKey` 的值，这里我们使用test环境，拷贝`test`的 key值  `2ZKF6OuLvf0BPhOKfs0b3IiSHjFJEk2EyF5--`到那里去，这里就要注意不同环境下的key值不一样，所以注意区分，最好写脚本去构建不同的包。
    再确保 **Bundle Version String， short 这一行的值是 1.0.0  而不是 1.0**，这个版本号也是我们后面更新时要用到（就是说：Code-Push支持针对不同的APP版本去做热更新）
    
        到这一步需要考虑的问题：
        
        1. 何时检查更新（每次启动时？每次更新时？特定时候：比如按一个按钮？）
        2. 何时安装更新（每次启动时？每次更新下载后就安装？提示用户安装？）
    这个是我们去考虑的问题

    在JavaScript文件里加入
    
    ```js
    var CodePush = require(“react-native-code-push”);
    //componentDidMount函数里加入
    CodePush.sync();
    ```



## CodePush更新过程
1. 打包更新文件  
    ① 仅更新js文件  
    
        ```
        tutumagideMBP:APPReact tufei$ react-native bundle --parameter ios --entry-file index.ios.js --bundle-output ./testCodePush/APP_task010001.js
        [16:09:02] <START> Building Dependency Graph
        [16:09:02] <START> Crawling File System
        [16:09:02] <START> find dependencies
        [16:09:05] <END>   Crawling File System (3349ms)
        [16:09:05] <START> Building in-memory fs for JavaScript
        [16:09:05] <END>   Building in-memory fs for JavaScript (166ms)
        [16:09:05] <START> Building in-memory fs for Assets
        [16:09:05] <END>   Building in-memory fs for Assets (118ms)
        [16:09:05] <START> Building Haste Map
        [16:09:05] <START> Building (deprecated) Asset Map
        [16:09:05] <END>   Building (deprecated) Asset Map (40ms)
        [16:09:05] <END>   Building Haste Map (105ms)
        [16:09:05] <END>   Building Dependency Graph (3776ms)
        transformed 552/552 (100%)
        [16:09:06] <END>   find dependencies (3898ms)
        bundle: start
        bundle: finish
        bundle: Writing bundle output to: ./testCodePush/APP_task010001.js
        bundle: Done writing bundle output
        Assets destination folder is not set, skipping...
        ```

    ② 更新js和图片
    
        ```
        tutumagideMBP:APPReact tufei$ react-native bundle --parameter ios --entry-file index
        .ios.js --bundle-output ./bundles/SwitchCheck010004.js --assets-dest ./bundles
        [20:21:50] <START> Building Dependency Graph
        [20:21:50] <START> Crawling File System
        [20:21:51] <START> find dependencies
        [20:21:55] <END>   Crawling File System (4440ms)
        [20:21:55] <START> Building in-memory fs for JavaScript
        [20:21:55] <END>   Building in-memory fs for JavaScript (199ms)
        [20:21:55] <START> Building in-memory fs for Assets
        [20:21:55] <END>   Building in-memory fs for Assets (128ms)
        [20:21:55] <START> Building Haste Map
        [20:21:55] <START> Building (deprecated) Asset Map
        [20:21:55] <END>   Building (deprecated) Asset Map (47ms)
        [20:21:55] <END>   Building Haste Map (115ms)
        [20:21:55] <END>   Building Dependency Graph (4907ms)
        transformed 552/552 (100%)
        [20:21:56] <END>   find dependencies (4677ms)
        bundle: start
        bundle: finish
        bundle: Writing bundle output to: ./bundles/SwitchCheck010004.js
        bundle: Done writing bundle output
        bundle: Copying 5 asset files
        bundle: Done copying assets
        ```
    **注意**
    >这里打包更新的文件时，并不是Code-push给我们做的，是React-native框架给我们完成的，当我们执行上面指令时，其实是执行了`node node_modules/react-native/local-cli/cli.js bundle --assets-dest ./bundles/ --bundle-output ./bundles//main.jsbundle --dev true --entry-file index.ios.js --platform ios `类似的指令，可以自己在本地执行一下，打包的资源在 --assets-dest 指定的目录以及打包的js文件在 --bundle-output 指定的文件，Code-Push只是默默帮我们封装了一层并且压缩上传。可以输入--dev true 参数，观察目录变化，当出现.zip文件时，立刻解压（因为Code-push在上传成功后会删掉此文件），解压出来的就是我们上面执行指令的生成文件。
2. 发布更新文件

    `code-push relase [AppName] -d [Deployment参数] [更新文件目录] [需要更新到App的版本号]`
    
    ```
    tutumagideMBP:APPReact tufei$ code-push release APP_task -d test ./testCodePush/APP_task010001.js 1.0.0
    Upload progress:[==================================================] 100% 0.0s
    Successfully released an update containing the "./testCodePush/APP_task010001.js" file to the "test" deployment of the "APP_task" app.
    ```

3. 更好用的命令

    ```
    //综合的上面说的1，2步的操作：1，新建存放打包后的目录，打包js和资源文件到此目录；2，根据命令行配置参数更新此目录的内容
    code-push release-react <appName> <platform>
    [--bundleName <bundleName>]
    [--deploymentName <deploymentName>]
    [--description <description>]
    [--development <development>]
    [--disabled <disabled>]
    [--entryFile <entryFile>]
    [--mandatory]
    [--sourcemapOutput <sourcemapOutput>]
    [--targetBinaryVersion <targetBinaryVersion>]
    [--rollout <rolloutPercentage>]
    //官方文档对参数都有说明
    ```
    发布后我们的App就可以根据自己的改动以及更新机制去更新了。

## 总结
根据参数说明可以知道Code-Push支持的配置以及功能有  

1. deploymentName：根据环境配置（test，dev，product）
2. mandatory：是否强制更新（检测到更新，下载后立即运行）
3. rollout：灰度更新
4. code-push rollback <appName> <deploymentName> 回滚
5. code-push promote <appName> <sourceDeploymentName> <destDeploymentName>
[--description <description>]
[--disabled <disabled>]
[--mandatory]
[--rollout <rolloutPercentage>]
[--targetBinaryVersion <targetBinaryVersion]
测试的更新可以直接提升为生成环境的更新，不用再次打包更新（避免测试与发布不一致的问题）

6. 版本可追踪
code-push deployment history <appName> <deploymentName>

7. 更新的安装情况可知

```
tufei$ code-push deployment list  APP_task
┌────────────┬──────────────────────────────────┬───────────────────────┐
│ Name       │ Update Metadata                  │ Install Metrics       │
├────────────┼──────────────────────────────────┼───────────────────────┤
│ dev        │ No updates released              │ No installs recorded  │
├────────────┼──────────────────────────────────┼───────────────────────┤
│ Production │ No updates released              │ No installs recorded  │
├────────────┼──────────────────────────────────┼───────────────────────┤
│ Staging    │ No updates released              │ No installs recorded  │
├────────────┼──────────────────────────────────┼───────────────────────┤
│ test       │ Label: v3                        │ Active: 100% (1 of 1) │
│            │ App Version: 1.0.0               │ Total: 1              │
│            │ Mandatory: No                    │                       │
│            │ Release Time: 9 hours ago        │                       │
│            │ Released By: tufei.fly@gmail.com │                       │
└────────────┴──────────────────────────────────┴───────────────────────┘
```
**更多的功能大家自行安装体验便知。**
## 参考
1. [React-native-code-push README](https://github.com/Microsoft/react-native-code-push#getting-started)  
2. [CodePush Management](https://github.com/Microsoft/code-push/blob/master/cli/README.md)  
3. 部分摘自[使用者的经验](http://blog.csdn.net/oiken/article/details/50279871)，如有侵权，请于[联系](tufei.fly@gmail.com)  

注：Code-Push体验版本: **1.10.0-beta**，仅在iOS上测试通过
