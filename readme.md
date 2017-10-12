## 环境准备
- git
- npm
- hexo-cli (npm install -g hexo-cli)
- hexo-deployer (npm install -g hexo-deployer-git)

## 还原
```
npm install
```

## 常用命令

```
//新建文章
hexo new "artical name"

//新建草稿
hexo new draft "artical name"

//发布草稿
hexo publish "artical name"

//启动本地服务器（草稿预览）：
hexo server -p 3000 --drafts

//清空缓存：
hexo clean

//生成静态页面：
hexo generate

//部署到github：
hexo deploy
```

## (Optional) 修改hexo端口（默认4000被占用）

找到node_modules\hexo-server\index.js文件，可以修改默认的port值
