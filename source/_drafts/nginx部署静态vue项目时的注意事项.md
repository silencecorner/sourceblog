---
title: nginx部署静态vue项目时的注意事项
tags:
- vue
- nginx
categories:
- 前端
- 运维
---
## 问题
在使用webpack或者vuecli3作为脚手架开发vue项目时，使用内置的express编写测试代码都没有问题，运维拿到build生成的dist文件之后，需要通过这样方式访问`http:/xxxx.com/content-path`。如何部署才能保证前端页面资源正确加载呢？
## 解决
### root部署
假设content-path=`post`，运维同学要rename `dist`文件夹为`post`，上级文件目录`C:\Users\silencecorner\Project\graphql-grpc-exmaple\vue-apollo-sample`
```bash
mv dist post
```
配置nginx.conf应该如下
```conf
location / {
    root   C:\Users\silencecorner\Project\graphql-grpc-exmaple\vue-apollo-sample;
    index  index.html index.htm;
}
```
成功访问到`http://localhost/post/`

---
{% asset_img 测试.png 测试结果 %}
### alias部署
以root部署例子为前提,那么我们的配置文件nginx.conf应该如下
```conf
location /post {
    alias   C:\Users\silencecorner\Project\graphql-grpc-exmaple\vue-apollo-sample\dist;
    index  index.html index.htm;
}
```
记得执行一下`.\nginx.exe -s reload`,这是访问我们的`http://localhost/post/`

---
{% asset_img alias测试1.png alias测试1结果 %}
如图访问css资源`http://localhost/css/app.e2707afb.css` 404啦，我们怎样去给这个连接下加个`post`呢？以vuecli3为例,修改`vue-config.js`添加`publicPath`指定值为`post`
```
console.log(`当前graphql网关访问地址:${process.env.VUE_APP_GRAPHQL_HTTP}`)
module.exports = {
    publicPath: 'post',
    pluginOptions: {
        graphqlMock: false,
        apolloEngine: false,
    },
    devServer: {
        port: 8081,
    }
}
```
此时重新打包再次访问就能正常访问啦！

## 总结
这里有两种部署方式，运维上来讲更倾向于第二种，前端可以配置使用`process.env.XXX`变来获取控制台变量，运维打包时想部署到任何路径都可以啦！当然使用root方式也是能达到效果，但是可能会给运维同学造成困扰，遇到这个问题的前端同学就不要说在本地我们用`devServer`跑都没问题的这种话啦! 

---
have a nice day ^_^