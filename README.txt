# 1 安装
npm install -g hexo-cli
npm install hexo-deployer-git --save
npm install hexo-renderer-markdown-it
npm install hexo-renderer-markdown-it-plus
hexo init blog
# 修改blog/_config.yml的内容
# (1) source_dir: ../MyBlog
# (2) author: zhengchenyu
# (3) deploy:
#       type: git
#       repository: https://github.com/zhengchenyu/zhengchenyu.github.com.git
#       branch: master

# 2 发布博客
cd blog
hexo n "title"
# 修改../MyBlog
hexo clean
hexo g		# 生成
hexo s		# 启动本地server调试
hexo d		# 发布到github

# 3 支持数学公式
# 详见: https://blog.csdn.net/littlehaes/article/details/84370393

