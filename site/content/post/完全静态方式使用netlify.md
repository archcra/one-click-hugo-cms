---
title: 完全静态方式使用netlify
date: 2018-10-14T06:23:22.920Z
description: 使用全托管的方式固然好，但也可能受限制。完全自已控制的方式的静态网站，可能是我们所需要的。即完全不需要netlify网站的参与，是不是很清爽
---
上次说了关于\[空手套白狼](http://blog.nowhat.me/post/%E7%A9%BA%E6%89%8B%E5%A5%97%E7%99%BD%E5%84%BF%E7%8B%BC/)的方法，但是，总是有但是：

使用全托管的方式固然好，但也可能受限制。完全自已控制的方式的静态网站，可能是我们所需要的。即完全不需要netlify网站的参与，是不是很清爽？

我是希望使用这种方式的，不想为netlify所束缚。那如何呢？经过一翻尝试，发现github不行：因为github需要服务端验证，这个不适合于\*\*静态\*\*网站，当然在netlify配置一下能解决这个问题，不过前面说了，我们不准备依赖netlify或其它网站（当然不包括github或 gitlab了）。使用gitlab是有机会的，下面我简要说一下。

首先，需要在gitlab上\[开启一个oAuth的应用](https://docs.gitlab.com/ee/integration/oauth_provider.html#adding-an-application-through-the-profile),这里面，主要有两个信息：一是那个应用的ID，二是回调的域名。当然，这里虽然不需要别的网站，但是，需要自己有独立的域名，否则无法回调。至于回调有什么用，嗯，我现在也说不清楚。

然后就是在静态网站中，加一个目录，如admin，然后是两个文件：

admin

├ index.html

└ config.yml

相应的说明，可以从\[这里](https://www.netlifycms.org/docs/add-to-your-site/?no-cache=1#configuration)找到，然后访问这个admin，就可以使用gitlab帐户登录，进行信息的生产了。可以加入图片信息。不过，这些信息虽然进入gitlab没有问题，但是，信息的消费，如何进行？这点我到目前还没有搞清楚。只是查到了一个需求或bug，即配置中，目录是静态的，文件名称虽然是动态，但只支持一级目录，不支持二级目录：

\`\``

folder: "_posts/blog" # The path to the folder where the documents are stored

create: true # Allow users to create new documents in this collection

slug: "{{year}}-{{month}}-{{day}}-{{slug}}" # Filename template, e.g., YYYY-MM-DD-title.md

\`\``

如上的配置，希望的情形是：

\`\``

folder: "html/{slug}" # The path to the folder where the documents are stored

create: true # Allow users to create new documents in this collection

slug: "index"

\`\``

这样，会在相应的目录下，生成动态的二级目录。这种格式不但为其他人所期望，也是pico CMS的标准格式。如果能生成这样格式的数据，那么消费就容易解决了：使用pico CMS来展示。可是，到目前为止(2018.10.14)，这个需求应该还没有实现。

所以，读到这里，这条路，目前是不大通畅的。Sorry!
