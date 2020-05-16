---
title: Spring Security框架原理--OAuth2流程
date: 2019-05-31 15:37:34
categories: Spring Security
---
## 客户端模式（Client Credentials Grant）
![OAuth2 客户端模式](/images/springsecurity/OAuth2 客户端模式.png)

请求示例：/oauth/token?grant_type=client_credentials&client_id=client&client_secret=secret&scope=read
![OAuth2 客户端模式示例](/images/springsecurity/OAuth2 客户端模式示例.png)

## 密码模式（Resource Owner Password Credentials Grant）
![OAuth2 密码模式](/images/springsecurity/OAuth2 密码模式.png)

请求示例：/oauth/token?grant_type=password&client_id=password&client_secret=secret&scope=read&username=admin&password=admin
![OAuth2 客户端模式示例](/images/springsecurity/OAuth2 客户端模式示例.png)

## 简化模式（Implicit Grant）
![OAuth2 简化模式](/images/springsecurity/OAuth2 简化模式.png)

请求示例：/oauth/authorize?response_type=token&client_id=implicit&redirect_uri=https://www.baidu.com&scope=read
登录之后，返回：
https://www.baidu.com/#access_token=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbiIsInVzZXJfbmFtZSI6ImFkbWluIiwic2NvcGUiOlsicmVhZCJdLCJleHAiOjE1NTkwMjY3OTIsImF1dGhvcml0aWVzIjpbIlJPTEVfVVNFUiJdLCJqdGkiOiI5ODI0NWY0Yi0wMDg0LTRkODQtYWU4MS1iZjhhNzRjMzM4YzkiLCJjbGllbnRfaWQiOiJpbXBsaWNpdCJ9.IqM0kIrUHX6_U4Xmmpn6VLlPLP_4zhdT-LSEyL6fW8mBQzzGVQzNrEpkvFBJLji1sq-0ZInl9TuWge5Rj0OsXrVC7wZjtFUiaT-ZC1bhWZx_tnuiATw72IUk3betWdVliYBgs979gZE8NXB6MChOIuXCuH5sSI7bj6fbfz0-56ja9RFtboNefntcoflYpGUaIion8emL0qbY09SgR_1D9GKKUal-4rnJWTAPbsaGtWCjOAaYDaYNVS_jE194zyqodSW1s27IfsOoPdr3WQ2ebF83RmH3h9o2Gw-INvfLVAHAh0YccSgeXGBkNBruxYo2tUaO3XlV8RSea30fXqqfAA&token_type=bearer&expires_in=86399&jti=98245f4b-0084-4d84-ae81-bf8a74c338c9

## 授权码模式
![OAuth2 授权码模式](/images/springsecurity/OAuth2 授权码模式.png)

流程示例：
![OAuth2 授权码模式示例](/images/springsecurity/OAuth2 授权码模式示例.png)

## 备注
* 四种模式，默认都要求用户授权，通过设置`autoApprove`为true，可以实现自动授权。
* 四种模式，默认都要求用户登录。客户端模式|密码模式下，通过配置`security.allowFormAuthenticationForClients()`，可以跳过登录步骤（使用`ClientCredentialsTokenEndpointFilter`）。
* 客户端模式|密码模式，直接请求到`TokenEndpoint`；授权码模式，先请求到`AuthorizationEndpoint`，然后转发请求到`TokenEndpoint`；简化模式，只会请求到`AuthorizationEndpoint`。
* 如果未指定scope，则scope为client的所有scopes。
* 密码模式|授权码模式，支持refresh token。客户端模式，通过设置`ClientCredentialsTokenGranter.allowRefresh`为true，可以返回refresh token。
* 应用场景：
    + 客户端模式：为后台api服务消费者设计
    + 密码模式：为遗留系统设计
    + 简化模式：为web浏览器应用设计
    + 授权码模式：标准模式

## TokenEndpoint
![TokenEndpoint 时序图1](/images/springsecurity/TokenEndpoint 时序图1.png)
![TokenEndpoint 时序图2](/images/springsecurity/TokenEndpoint 时序图2.png)
![TokenEndpoint 时序图3](/images/springsecurity/TokenEndpoint 时序图3.png)
![TokenEndpoint 时序图4](/images/springsecurity/TokenEndpoint 时序图4.png)
    
## AuthorizationEndpoint
![AuthorizationEndpoint 时序图1](/images/springsecurity/AuthorizationEndpoint 时序图1.png)
![AuthorizationEndpoint 时序图2](/images/springsecurity/AuthorizationEndpoint 时序图2.png)




