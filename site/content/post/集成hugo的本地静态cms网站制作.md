---
title: 集成Hugo的本地静态CMS网站制作
date: 2018-10-17T11:38:36.534Z
description: |
  这次要放大招了
---
```
本帖目标：gitlab集成的纯本地网站（除gitlab外，不依赖于其它网站资源）
需要了解：gitlab web hook, docker, nodejs, pm2, nginx
需要资源：服务器，域名，gitlab帐号
新鲜玩意儿：Hugo Themes，NETLIFY CMS
```

# 第一步，准备网站的基础资源 - 静态部份
## A. 使用docker容器来生成网站目录
```
docker run --rm -it -v $PWD:/src -u hugo jguyomard/hugo-builder hugo new site mysite
```
## B. 加入主题
```
cd mysite

# Now, you probably want to add a theme (see https://themes.gohugo.io/):
git init
git submodule add https://github.com/budparr/gohugo-theme-ananke.git themes/ananke;
echo 'theme = "ananke"' >> config.toml
```
## C.加入一个新帖子
```
docker run --rm -it -v $PWD:/src -u hugo jguyomard/hugo-builder hugo new posts/my-first-post.md

```
## D. 测试运行
```
docker run --rm -it -v $PWD:/src -p 1313:1313 -u hugo jguyomard/hugo-builder hugo server -w --bind=0.0.0.0
```
这时，使用浏览器访问本地的1313端口，就可以看到网站内容了。 


# 第二步 加入netlify
这步说难也难，说简单也简单。说难，是需要配置后端，即gitlab或github等。

## A. 难的部份：配置gitlab

首先，需要在gitlab上[开启一个oAuth的应用](https://docs.gitlab.com/ee/integration/oauth_provider.html#adding-an-application-through-the-profile),这里面，主要有两个信息：一是那个应用的ID，二是回调的域名。当然，这里虽然不需要别的网站，但是，需要自己有独立的域名，否则无法回调。至于回调有什么用，嗯，我现在也说不清楚。

## B. 说简单，就是一个目录，两个文件。
这一个目录、两个文件是：
```
admin
 ├ index.html
 └ config.yml
```
在这里，有[详细的说明](https://www.netlifycms.org/docs/add-to-your-site/)

配置中，app_id就是上一步设置oAuth里的id。
实际的例子，在[这里](https://gitlab.com/holibut/netlifycms/tree/master/static/admin)有。


之所以例子中有一个lib目录，是为了避免其中的库资源（netlify-cms.js
）在公网上访问不到，而保存到了本地。


# 第三步 部署服务
## A. 使用git pull从gitlab上获取代码
在服务器上，使用git从gitlab上得取资源。这步要使用ssh方式，配置gitlab上此服务器的部署服务器，即有信任关系。或者说，在执行git pull时，不需要输入用户、密码什么的。

## B. 启动服务
当然使用docker了，其实刚才在本地测试时，已使用过了，只不过这次改为服务式：
```
docker run -d --name netlifycms -v $PWD:/src -p 1313:1313 -u hugo jguyomard/hugo-builder hugo server -w --bind=0.0.0.0
```

## C. 配置nginx
由于gitlab的验证只能是https,所以nginx我们必须配置为https形式的。

## D. 配置gitlab的回调
这一步是用于当netlify对内容有变更时，会产生gitlab的push事件。这时我们需要响应这个事件，以更新服务器上的本地代码。这里有趣的是，这里的服务器，成了本地了。这个本地，是相对于gitlab来讲的：这个服务器，成了代码backend的客户端：即所谓的本地。

在项目的settings -> Integrations中，可以设置回调的地址：
```
https://gitlab.com/holibut/netlifycms/settings/integrations
```
如设置成这样：
```
http://gitlab-hook.cms.nowhat.me/hook
```
注意要点下面的Add webhook按钮。注意：这个地址要响应http post请求的。下面我就来说这个地址的服务如何实现。

## E. 响应webhook请求
其实这一步我一直没有把握：总感觉应该有更好的办法。但可惜我目前还没有找到。只好自己写了一个程序来实现这个功能。不过还好，程序是node.js程序，挺简单的。可以使用pm2去管理运行。
由于程序较小，我是嵌在其他项目下的，所以没有开源。下面把主配置及代码这到这里：
### package.json:
```
{
  "name": "gitlab-hook",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "express": "^4.16.4",
    "pug": "^2.0.3",
    "shelljs": "^0.8.2"
  }
}
```
### server.js
```
const express = require("express");
var shell = require('shelljs');

const app = express()

app.set('view engine', 'pug');

app.get('/', (req, res) => {
    res.render('index')
})

app.post('/hook', (req, res) => {
  shell.cd('/opt/apps/netlifycms');
  if (shell.exec('git pull').code !== 0) {
  shell.echo('Error: Git commit failed');
  shell.exit(1);
}
       console.log('pull done.')
       res.send('I received.')
   })

app.listen(5656, () => {
    console.log('http://localhost:5656')
})
```


注意pug的view是不需要的，如果不熟悉pug, 相应渲染的那句改成这样就可以了：
res.send('Hello World')

当然，整个get路由都是可以删除的。

# 尾声
好了，到了这里，服务已就绪了。
访问netlifycms目录，就可以看到hugo网站的内容。访问其下的admin目录，就可以使用netlify cms进行cms内容的维护。维护的内容会自动提交到gitlab中。然后通过我们配置的钩子，会自动更新到服务器上，这时docker的hugo服务会自动监听变动，以更新内容：即新的cms内容会被用户看到。好了，一个文件式cms，就这样愉快地完成了！
