## 环境准备
- git
- npm
- hexo-cli (npm install -g hexo-cli)
- hexo-deployer ()

## 还原
```
npm install
```

## 常用命令

```
//启动本地服务器： 
hexo server

//清空缓存：
hexo clean

//生成静态页面： 
hexo generate

//部署到github：  
hexo deploy
```

## 修改hexo端口（默认4000被占用）

找到node_modules\hexo-server\index.js文件，可以修改默认的port值

