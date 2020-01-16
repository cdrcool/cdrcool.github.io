---
title: Hexo 配置指南
date: 2020-01-15 10:51:00
categories: Hexo
---
## 修改站点信息
打开 \_config.yml 文件，修改其中的 Site 下的 title 等属性：
```yaml
# Site
title: 雨中的了悟
subtitle: ''
description: ''
keywords:
author: 晴天
language: zh-CN
timezone: ''
```

## 更换主题
Hexo 默认使用 landscape 主题，同时支持配置为其它主题，这里以配置为 next 主题来介绍如何操作。

1. 访问 [https://github.com/theme-next/hexo-theme-next](https://github.com/theme-next/hexo-theme-next)
2. 在根目录下执行命令：`git clone https://github.com/theme-next/hexo-theme-next themes/next`
3. 打开 \_config.yml 文件，将其中的 theme 属性由 landscape 改为 next

next 主题默认只开放了 首页 和 归档 两个菜单，且在侧边栏中只有 日志 链接可以点击查看详情，现在我想添加 分类 菜单，同时让侧边栏中的 分类 和 标签 链接都能点击。

### 添加分类菜单
打开 \themes\next\_config.yml 文件，将其中的 menu 属性下的 categories 属性前面的注释取消掉。

### 创建分类和标签链接
首先，执行以下两行命令：
```bash
hexo new page "categories"
hexo new page "tags"
```

之后，可以发现在 source 目录下多出了 categories 和 tags 两个子目录，然后分别修改这两个子目录中的 index.md 文件：
* categories
```
type: categories
date: 2020-01-15 11:24:00
comments: false
```

* tags
```
type: tags
date: 2020-01-15 11:24:00
comments: false
```

## 部署到 GitHub
1. 访问 [https://github.com/hexojs/hexo-deployer-git](https://github.com/hexojs/hexo-deployer-git)
2. 在根目录下执行命令：`npm install hexo-deployer-git --save`
3. 打开 \_config.yml 文件，在文件末尾添加以下配置：
```yaml
deploy:
  type: git
  repo: https://github.com/cdrcool/cdrcool.github.io
  branch: master
```
4. 执行命令：`hexo d -g`

## 集成 Gitalk
1. 访问 [https://github.com/settings/developers](https://github.com/settings/developers)
2. 创建 OAuth App：
![OAuth App 创建示例](/images/hexo/OAuth-App创建示例.png)
3. 打开 \themes\next\_config.yml 文件，配置 gittalk 相关属性：
```yaml
# Gitalk
# For more information: https://gitalk.github.io, https://github.com/gitalk/gitalk
gitalk:
  enable: true
  github_id: cdrcool # GitHub repo owner
  repo: cdrcool.github.io # Repository name to store issues
  client_id: 5f0b1c1d42822204f803 # GitHub Application Client ID
  client_secret: 56818d47fc75f1f4999acd7a3356879bbade310f # GitHub Application Client Secret
  admin_user: cdrcool # GitHub repo owner and collaborators, only these guys can initialize gitHub issues
  distraction_free_mode: true # Facebook-like distraction free mode
  # Gitalk's display language depends on user's browser or system environment
  # If you want everyone visiting your site to see a uniform language, you can set a force language value
  # Available values: en | es-ES | fr | ru | zh-CN | zh-TW
  language: zh-CN
```

注意：文件名中中英文之间不能带有空格，不然授权成功后回调时会报 redirect_uri_mismatch 异常。

## 启用打赏
打开 \themes\next\_config.yml 文件，配置 reward 相关属性：
```yaml
# Reward (Donate)
# Front-matter variable (unsupport animation).
reward_settings:
  # If true, reward will be displayed in every article by default.
  enable: true
  animation: true
  comment: 小礼物走一走，来 Github 关注我

reward:
  wechatpay: /images/wechatpay.png
  alipay: /images/alipay.png
  #bitcoin: /images/bitcoin.png
```

## 问题
### 大小写导致404
打开 .deploy_git\.git\config，将 ignorecase 属性的值由 true 改为 false，然后删除 .deploy_git 下所有文件，最后重新生成后部署：
```bash
hexo clean
hexo deploy -generate
```

### Gitment
#### Error: Not Found
检查 github_user 和 github_repo 是否配置正确
#### Error: Validation Failed
打开 \themes\next\layout\_third-party\comments\gitment.swig，找到以下代码，将id值由 `window.location.pathname` 改为 `decodeURI(window.location.pathname)`
```js
function renderGitment() {
    var gitment = new {{ CommentsClass }}({
      id: window.location.pathname,
      owner: '{{ theme.gitment.github_user }}',
      repo: '{{ theme.gitment.github_repo }}',
      {% if theme.gitment.mint %}
        lang: '{{ theme.gitment.language }}' || navigator.language || navigator.systemLanguage || navigator.userLanguage,
      {% endif %}
      oauth: {
      {% if theme.gitment.mint and theme.gitment.redirect_protocol %}
        redirect_protocol: '{{ theme.gitment.redirect_protocol }}',
      {% endif %}
      {% if theme.gitment.mint and theme.gitment.proxy_gateway %}
        proxy_gateway: '{{ theme.gitment.proxy_gateway }}',
      {% else %}
        client_secret: '{{ theme.gitment.client_secret }}',
      {% endif %}
        client_id: '{{ theme.gitment.client_id }}'
      }
    });
    gitment.render('gitment-container');
  }
```