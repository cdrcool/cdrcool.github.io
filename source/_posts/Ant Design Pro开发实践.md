---
title: Ant Design Pro 开发实践
date: 2020-03-17 14:37:00
categories: React
tags:
    - React
    - Ant Design
    - Ant Design Pro
---
## 开始使用
### 前序准备
本地环境需要安装 yarn、node 和 git。我们的技术栈基于 ES2015+、React、UmiJS、dva、g2 和 antd，提前了解和学习这些知识会非常有帮助。

### 安装
新建一个空的文件夹作为项目目录，并在目录下执行：

```yaml
yarn create umi
```

选择 ant-design-pro：

```bash
Select the boilerplate type (Use arrow keys)
❯ ant-design-pro  - Create project with an layout-only ant-design-pro boilerplate, use together with umi block.
  app             - Create project with a simple boilerplate, support typescript.
  block           - Create a umi block.
  library         - Create a library with umi.
  plugin          - Create a umi plugin.
```

Ant Design Pro 脚手架将会自动安装。

### 目录结构
Ant Design Pro 已经为我们生成了一个完整的开发框架，提供了涵盖中后台开发的各类功能和坑位，下面是整个项目的目录结构。

```text
├── config                   # umi 配置，包含路由，构建等配置
├── mock                     # 本地模拟数据
├── public
│   └── favicon.png          # Favicon
├── src
│   ├── assets               # 本地静态资源
│   ├── components           # 业务通用组件
│   ├── e2e                  # 集成测试用例
│   ├── layouts              # 通用布局
│   ├── models               # 全局 dva model
│   ├── pages                # 业务页面入口和常用模板
│   ├── services             # 后台接口服务
│   ├── utils                # 工具库
│   ├── locales              # 国际化资源
│   ├── global.less          # 全局样式
│   └── global.tsx           # 全局 JS
├── tests                    # 测试工具
├── README.md
└── package.json
```

### 本地开发
安装依赖：

```bash
yarn install
```

启动：
```bash
yarn start
```

启动完成后会自动打开浏览器访问 localhost:8000，看到下面的页面就代表成功了。

![Ant Design Pro预览](../images/react/Ant%20Design%20Pro预览.png)

接下来我们可以修改代码进行业务开发了，Ant Design Pro 内建了模拟数据、HMR 实时预览、状态管理、国际化、全局路由等等各种实用的功能辅助开发，我们可以继续阅读和探索其他文档。

## 个性化调整
### 更换地址栏 logo和标题
更改 logo：
该 logo 位于 /public/favicon.svg，可以用其它 logo 将其替换掉，或者修改 /src/pages/document.ejs 中的 link 元素的 href 属性。

更换标题：
修改 /src/pages/document.ejs 中的 title 内容。

### 更换主页 logo 和标题
更换 logo：
该 logo 位于 /src/assets/logo.svg，可以用其它 logo 将其替换掉，或者修改 BasicLayout.tsx 中的 ProLayout 组件的 logo 属性。

更换标题：
修改 /config/defaultSetting.ts 中的 title 属性。

### 更换启动时 logo 和标题
修改 /src/pages/document.ejs 中第 2 个 img 元素的 src 属性和紧跟 img 后的内容。

### 更换首页刷新时 logo
该 logo 位于 /public/pro_icon.svg，可以用其它 logo 将其替换掉，或者修改 /src/pages/document.ejs 中的第 1 个 img 元素的 src 属性。

### 修改页脚内容
修改 /src/layouts/BasicLayout.tsx 中的 DefaultFooter 组件的 copyright 和 links 属性。

### 更换主题
Ant Design Pro 默认提供了 dark(默认) 和 light 这两种主题，可以通过修改 /config/defaultSetting.js 中的 navTheme 属性来更换主题。

### 更换导航模式
Ant Design Pro 同时支持侧边栏和顶部栏显示导航，可以通过修改 /config/defaultSetting.js 中的 layout 属性来更换导航模式。

### 移除头部不必要的小组件
在实际开发过程中，我们的项目可能不需要 "全局搜索"、"使用文档" 、"国际化切换"这些组件，可以到 /src/components/GlobalHeader/RightContent.js 中注释掉 HeaderSearch、Tooltipown、SelectLang 这几个组件的使用。

### 显示个人中心/设置菜单
在 /src/components/GlobalHeader/RightContent.tsx 中引用 AvatarDropdown 组件的地方添加 menu 属性。

### 显示通知/消息/代办
在 /src/components/GlobalHeader/RightContent.js 中引入并使用 NoticeIconView 组件。

### 调整用户登录页
对于用户登录页的 logo、标题、描述、登录方式、页脚和国际化支持，也可以到 /src/layouts/UserLayout.tsx 中做相应调整。

### 不显示页面 title
每个页面都由 PageHeaderWrapper 封装，PageHeaderWrapper 默认会显示页面 title，但是 title 已经在面包屑中显示了，因此可以隐藏掉 PageHeaderWrapper 中的 title，具体做法是给其传递 title = false。

### 未登录跳转到登录页
默认情况下，用户未登录也会跳转到欢迎页，如果要实现未登录时跳转到登录页，只需在 /config/config.js 中的 routes 列表里的 ../layouts/SecurityLayout 那一项中添加 Routes: \['src/pages/Authorized']。
还有一点需要注意，默认在用户退出登录后，Ant Design Pro 并没有清理掉用户权限信息，我们需要在 /src/models/login.ts 的 logout 方法中添加 localStorage.removeItem('antd-pro-authority') 。

### 不启动 Umi UI
```bash
yarn start:no-ui
```

### 关闭 Mock 数据
1. 启动命令改为：`yarn start:no-mock`。
2. 修改 /config/proxy.ts：
```text
...
dev: {
    '/api/': {
      target: 'http://localhost:8080/',
      changeOrigin: true,
      pathRewrite: {'^/api/': ''},
    }
}
...
```

注意：查看控制台，会发现请求的地址还是 localhost:8000/，但实际上已经请求到后端地址了。

