# Let's Build a Simple Database

[View rendered tutorial](https://cstack.github.io/db_tutorial/) (with more details on what this is.)

## 备忘
安装 Ruby 和 Jekyll：[https://jekyllrb.com/docs/installation/](https://jekyllrb.com/docs/installation/)

在本地调试：
```
# 第一次跑先安装依赖
## 可能需要先设置一下依赖的目录，不然可能会遇到写入权限问题
bundle config set --local path 'vendor/bundle'
bundle install

# 构建
bundle exec jekyll build

# 启动
bundle exec jekyll serve
```