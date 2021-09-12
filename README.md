# 13 天自制简易数据库

[在线预览](https://db-tutorial.yian.me/) | 原作：[https://github.com/cstack/db_tutorial](https://github.com/cstack/db_tutorial) | [原版（英文）在线预览](https://cstack.github.io/db_tutorial/) 

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