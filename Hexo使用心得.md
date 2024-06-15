需要注意node.js和npm版本与操作系统是否匹配

部署到本地服务器的方法：
    1、下载选择好的网站包
    2、进入到网站目录下，执行npm install hexo-cli -g和npm install命令
    3、进入到网站包根目录中，执行hexo generate命令，生成public静态文件包
    4、进入到public目录下，执行hexo server命令，启动本地服务器

将进程放至后台与ssh连接进程脱离绑定的方法：
    1、在命令行中输入 screen -S blog 并回车，这会创建一个新的 screen 会话，名叫 'blog'
    2、在新的 screen 会话中输入 hexo server 并回车，这会在 'blog' 会话中启动 Hexo 服务
    3、按 Ctrl+A 然后 D，这样你就可以将 'blog' 会话切换到后台并返回到你的原始 SSH 会话中
    4、如果你想要回到 'blog' 会话，只需输入 screen -r blog 即可