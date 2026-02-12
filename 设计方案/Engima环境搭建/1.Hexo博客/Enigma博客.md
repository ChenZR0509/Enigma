---
title: Enigma博客
series: Enigma
categories:
  - 个人项目
  - EnigmaBlog
tags:
  - HEXO博客
abbrlink: 36239
date: 2023-05-10 16:40:00
---

# Enigma博客

>**ChenZR：**
>
>>2023年5月10日：我喜欢一边学习一边做笔记，这样既可以梳理清楚知识间的脉络层次关系，也可以在遗忘后及时复习、查缺补漏。我一直希望能有一款笔记软件能够兼顾简洁、免费、云同步、专业公式编辑等优点，但是市面上的软件总体表现都差强人意。正好，趁着近期毕业闲来无事，想着搭建一个属于自己的博客网站，在网站上发表一些自己的学习记录。

------

## 系列文章

{% series Enigma系列 %}

---

## 第一部分	项目配置

### 第一章	Windows电脑

- 相关软件：

  >- NodeJs：开源和跨平台的 JavaScript 运行环境
  >- Hexo：博客框架
  >- Git：代码版本管理工具
  >- WebStorm：代码编辑工具
  >- Putty：服务器链接工具
  >- WinSCP：服务器文件传输工具

#### 第一节	NodeJs的安装

- 进入[Node.js — 在任何地方运行 JavaScript (nodejs.org)](https://nodejs.org/zh-cn)这个页面，选择对应自己电脑系统(windows或mac)的安装包进行下载与安装。

  >**ChenZR：**正常下载安装即可，安装完成后打开命令行窗口，验证是否安装成功。

  ```shell
  npm -v
  node -v
  ```

- 在Nodejs安装目录下新建两个目录node_global和node_cache，并在以管理员身份打开命令行窗口：

  >**ChenZR：**后面下载的全局NodeJs安装包和缓存都会安装在这两个目录下

  ```shell
  #全局安装
  npm install -g <package_name>
  #路径需要根据自己实际情况自行更改
  npm config set prefix "E:\SoftwareMust\Nodejs\node_global"
  npm config set cache "E:\SoftwareMust\Nodejs\node_cache"
  ```

- 环境变量配置：

  >**ChenZR：**可以在任意路径下使用NodeJs插件的命令

  - 在系统的Path变量中添加：

    ```
    E:\SoftwareMust\Nodejs\node_global\node_modules
    ```

#### 第二节	Git的安装

- 进入后面的地址下载git工具[Git - Downloading Package (git-scm.com)](https://git-scm.com/download/win)

  >**ChenZR：**默认安装即可，安装完成后打开GitBash，对git用户进行配置。

  ```shell
  git config --global user.name "ChenZR"
  git config --global user.email "ChenZR20010509@outlook.com"
  git config user.name
  git config user.email
  #创建SSH
  ssh-keygen -t rsa -C "ChenZR20010509@outlook.com"
  ```

#### 第三节	Hexo博客创建

- NodeJs安装Hexo插件：

  ```shell
  npm install hexo-cli -g
  ```

- 在磁盘中确定好博客项目的路径，让命令行窗口切换到项目路径下，进行初始化操作：

  ```shell
  cd #自己的博客路径
  hexo init Blog	#hxeo初始化：在自己的路径下创建博客的基本运行文件
  cd Blog	#进入到博客路径下
  npm install #下载相应的nodejs插件
  npm install hexo-deployer-git --save #下载git插件
  hexo server	#hexo生成本地网页，当源文件进行更改时，刷新网页即可 
  ```

- 主题配置：

  ```shell
  #cd到Blog的根目录
  git clone -b master https://github.com/jerryc127/hexo-theme-butterfly.git themes/butterfly
  #在blog/_config.yml文件中，配置主题
  theme: butterfly
  #安装butterfly所需插件：
  npm install hexo-renderer-pug hexo-renderer-stylus --save
  ```

- Git仓库创建：

  >**ChenZR：**首先需要创建自己的github账号，然后创建博客仓库，这样即使后面更换电脑，那么整个项目也可以快速恢复。

  1. 把theme路径下butterfly目录拷贝一份，并删除里面的“.git目录”、“.gitignore目录”和".github目录"

  2. 把之前创建的项目删了然后重现参考上面的步骤创建一个博客项目，不要git下载主题，直接把上一步骤拷贝的主题文件复制到相关目录下。

     >**ChenZR：**可以把butterfly主题作为一个git子模块来处理，但是butterfly主题的更新会覆盖掉后面涉及到的图像资源和自定义配置文件，所以才进行这个操作。

  3. 使用WebStorm初始化Git仓库，然后提交和推送即可。

  4. 如果说想要恢复项目，那么从我们的仓库clone下来，然后WebStorm会自动提示你要进行npm install，按照操作进行即可。

- Hexo基本命令：

  ```shell
  #删除public文件的内容
  hexo clean
  #生成静态文件到public
  hexo generate
  hexo g
  #本地运行网页
  hexo server
  hexo s
  #发布静态文件
  hexo dploy
  hexo d
  ```

### 第二章	Linux服务器

#### 第一节	Ubuntu系统

- 软件安装：

  - build-essential：C/C++构建工具

    ```shell
    sudo apt install build-essential
    ```

  - openssl：提供SSL/TLS 和加密功能，可以做 HTTPS、加密通信等。

  - openssl-devel：openssl编译时需要的库

    ```shell
    sudo apt-get install openssl
    sudo apt-get install libssl-dev
    ```

  - libtool：库管理工具，帮助生成和管理共享库（.so文件），自动处理编译和链接。

    ```shell
    sudo apt-get install libtool
    ```

  - zlib：提供数据压缩/解压缩功能。

  - zlib-devel：提供 编译时用到的头文件和静态/共享库。

    ```shell
    sudo apt-get install zlib1g
    sudo apt-get install zlib1g.dev
    ```

  - pcre：是一个开源的正则表达式库，允许程序在 C/C++ 等语言中使用类似Perl 的正则表达式语法来进行字符串匹配和处理。

    ```shell
    cd /usr/local/src
    #下载源码
    wget http://downloads.sourceforge.net/project/pcre/pcre/8.37/pcre-8.37.tar.gz
    #解压
    tar -xvf pcre-8.37.tar.gz
    cd pcre-8.37
    ./configure
    #编译和下载
    make && make install
    #查看版本信息
    pcre-config --version
    ```

  - nginx：一个高性能的HTTP 和反向代理服务器，同时也可以用作邮件代理服务器（SMTP/POP3/IMAP）。

    ```
    sudo apt-get install nginx
    cd /usr/local/
    #下载源码
    wget http://nginx.org/download/nginx-1.17.9.tar.gz
    #解压
    tar -xvf nginx-1.17.9.tar.gz
    cd nginx-1.17.9
    ./configure
    #编译和下载
    make && make install
    ```

  - nodejs：一个开源的、跨平台的 JavaScript 运行时环境，它让 JavaScript 不再局限于浏览器中运行，而可以在服务器端运行。

    ```shell
    sudo apt install nodejs npm
    #查看版本信息
    nodejs --version
    npm -v
    ```

  - git：代码版本管理工具。

    ```shell
    sudo apt-get install git
    ```

- 博客仓库配置：

  - 创建一个用户，用户git的管理：

    ```shell
    #创建了一个git账户
    adduser git
    ```

  - 设置git的权限，使其能使用sudo命令：

    ```shell
    #将sudoers文件的权限设置为【rwx r-- ---】
    chmod 740 /etc/sudoers
    #配置sudo命令使得git用户可以使用sudo命令
    	#编辑/root/etc/sudoers
    	#找到以下内容
    	## Syntax:
    	##
    	## 	user	MACHINE=COMMANDS
    	##
    	## The COMMANDS section may have other options added to it.
    	##
    	## Allow root to run any commands anywhere 
    	root	ALL=(ALL) 	ALL
    	#在下面一行添加：使得git用户可以以超级用户的身份执行任何命令
    	git 	ALL=(ALL) 	ALL
    #将sudoers文件的权限设置为【r-- --- ---】
    chmod 400 /etc/sudoers
    ```

  - 切换到git用户配置ssh和git仓库：

    ```shell
    #切换git用户，这个用户名可以自己随便起
    su git
    ```

    - 配置SSH：

      ```shell
      cd ~
      mkdir .ssh
      cd .ssh
      #这里使用指令也可以使用WinSCP打开此文件进行操作
      #进入到文件内复制自己电脑C:\Users\15424\.ssh\id_rsa.pub文件内容
      vim authorized_keys
      #把id_rsa.pub文件内容粘贴到这个文件
      #添加文件权限
      #【rw- --- ---】
      chmod 600 ~/.ssh/authorized_keys
      #【rwx --- ---】
      chmod 700 ~/.ssh
      ```

    - 验证SSH有效性，打开自己电脑的命令行窗口：

      ```
      ssh -v git@服务器的公网ip
      ```

    - 配置博客仓库：

      ```shell
      #添加git项目上传目录
      cd ~
      git init --bare blog.git
      #打开此文件
      vi ~/blog.git/hooks/post-receive
      #在文件中添加
      git --work-tree=/home/www/website --git-dir=/home/git/blog.git checkout -f
      #添加文件可执行权限
      chmod +x ~/blog.git/hooks/post-receive
      ```

    - 验证git有效性：

      ```shell
      #更改hexo的配置文件：
      deploy:
        type: git
        repo: 创建的用户名@公网IP:/home/git/blog.git
        branch: master
      #git部署
      hexo clean
      hexo g -d
      #查看/home/www/website里面是否有内容，即可判断git是否有效
      ```

  - 配置nginx：

    - 创建网页：

      ```shell
      su root
      cd /home
      mkdir www
      cd www
      mkdir website
      chmod 777 /home/www/website
      chmod 777 /home/www
      ```

    - 配置nginx的映射：

      ```shell
      vim /usr/local/nginx/conf/nginx.conf
      #在其中找到root /var/www/html;并注释掉
      #添加下面的内容：
      root /home/www/website;
      ```

    - nginx控制命令：

      ```shell
      #启动
      service nginx start
      #停止
      service nginx stop
      #重启
      service nginx reload
      ```

- 博客验证：

  1. hexo博客部署：

     ```shell
     hexo clean
     hexo g -d
     ```

  2. 输入服务器公网IP，默认80端口，查看网页是否正常。

#### 第二节	CentOs系统

- 软件安装：

  - gcc：C语言编译器工具链

    ```shell
    yum install -y gcc 
    ```

  - gcc-c++：C++编译工具链

    ```shell
    yum install -y gcc-c++
    ```

  - make：自动化编译工具，根据Makefile来执行编译和安装步骤。

    ```shell
    yum install -y make
    ```

  - openssl：提供SSL/TLS 和加密功能，可以做 HTTPS、加密通信等。

  - openssl-devel：openssl编译时需要的库

    ```shell
    yum install -y openssl
    ```

  - libtool：库管理工具，帮助生成和管理共享库（.so文件），自动处理编译和链接。

    ```shell
    yum install -y libtool
    ```

  - zlib：提供数据压缩/解压缩功能。

  - zlib-devel：提供 编译时用到的头文件和静态/共享库。

    ```shell
    yum -y install zlib zlib-devel
    ```

  - pcre：是一个开源的正则表达式库，允许程序在 C/C++ 等语言中使用类似Perl 的正则表达式语法来进行字符串匹配和处理。

    ```shell
    cd /usr/local/src
    #下载源码
    wget http://downloads.sourceforge.net/project/pcre/pcre/8.37/pcre-8.37.tar.gz
    #解压
    tar -xvf pcre-8.37.tar.gz
    cd pcre-8.37
    ./configure
    #编译和下载
    make && make install
    #查看版本信息
    pcre-config --version
    ```

  - nginx：一个高性能的HTTP 和反向代理服务器，同时也可以用作邮件代理服务器（SMTP/POP3/IMAP）。

    ```shell
    cd /usr/local/
    #下载源码
    wget http://nginx.org/download/nginx-1.17.9.tar.gz
    #解压
    tar -xvf nginx-1.17.9.tar.gz
    cd nginx-1.17.9
    ./configure
    #编译和下载
    make && make install
    ```

  - nodejs：一个开源的、跨平台的 JavaScript 运行时环境，它让 JavaScript 不再局限于浏览器中运行，而可以在服务器端运行。

    ```shell
    curl -sL https://rpm.nodesource.com/setup_10.x | bash -
    yum install -y nodejs
    #查看版本信息
    node -v
    #查看版本信息
    npm -v
    ```

  - git：代码版本管理工具。

    ```shell
    yum install git
    ```

- 博客仓库配置：

  - 创建一个用户，用户git的管理：

    ```shell
    #创建了一个git账户
    adduser git
    ```

  - 设置git的权限，使其能使用sudo命令：

    ```shell
    #将sudoers文件的权限设置为【rwx r-- ---】
    chmod 740 /etc/sudoers
    #配置sudo命令使得git用户可以使用sudo命令
    	#编辑/root/etc/sudoers
    	#找到以下内容
    	## Syntax:
    	##
    	## 	user	MACHINE=COMMANDS
    	##
    	## The COMMANDS section may have other options added to it.
    	##
    	## Allow root to run any commands anywhere 
    	root	ALL=(ALL) 	ALL
    	#在下面一行添加：使得git用户可以以超级用户的身份执行任何命令
    	git 	ALL=(ALL) 	ALL
    #将sudoers文件的权限设置为【r-- --- ---】
    chmod 400 /etc/sudoers
    ```

  - 切换到git用户配置ssh和git仓库：

    ```shell
    #切换git用户，这个用户名可以自己随便起
    su git
    ```

    - 配置SSH：

      ```shell
      cd ~
      mkdir .ssh
      cd .ssh
      #这里使用指令也可以使用WinSCP打开此文件进行操作
      #进入到文件内复制自己电脑C:\Users\15424\.ssh\id_rsa.pub文件内容
      vim authorized_keys
      #把id_rsa.pub文件内容粘贴到这个文件
      #添加文件权限
      #【rw- --- ---】
      chmod 600 ~/.ssh/authorized_keys
      #【rwx --- ---】
      chmod 700 ~/.ssh
      ```

    - 验证SSH有效性，打开自己电脑的命令行窗口：

      ```
      ssh -v git@服务器的公网ip
      ```

    - 配置博客仓库：

      ```shell
      #添加git项目上传目录
      cd ~
      git init --bare blog.git
      #打开此文件
      vi ~/blog.git/hooks/post-receive
      #在文件中添加
      git --work-tree=/home/www/website --git-dir=/home/git/blog.git checkout -f
      #添加文件可执行权限
      chmod +x ~/blog.git/hooks/post-receive
      ```

    - 验证git有效性：

      ```shell
      #更改hexo的配置文件：
      deploy:
        type: git
        repo: 创建的用户名@公网IP:/home/git/blog.git
        branch: master
      #git部署
      hexo clean
      hexo g -d
      #查看/home/www/website里面是否有内容，即可判断git是否有效
      ```

  - 配置nginx：

    - 创建网页：

      ```shell
      su root
      cd /home
      mkdir www
      cd www
      mkdir website
      chmod 777 /home/www/website
      chmod 777 /home/www
      ```

    - 配置nginx的映射：

      ```shell
      vim /usr/local/nginx/conf/nginx.conf
      #在其中找到root /var/www/html;并注释掉
      #添加下面的内容：
      root /home/www/website;
      ```

    - nginx控制命令：

      ```shell
      #启动
      service nginx start
      #停止
      service nginx stop
      #重启
      service nginx reload
      ```

- 博客验证：

  1. hexo博客部署：

     ```shell
     hexo clean
     hexo g -d
     ```

  2. 输入服务器公网IP，默认80端口，查看网页是否正常。

## 第二部分	Hexo博客美化

### 第一章	Hexo配置

#### 第一节	基础配置

- 基础插件：

  ```shell
  #链接永久化插件
  npm install hexo-abbrlink --save
  #butterfly顶部菜单栏按钮
  npm install hexo-butterfly-extjs
  ```
  
- 基本信息：

  ```yaml
  #网页信息
  title: CzrTuringB的博客
  subtitle: 'CzrTuringB'
  description: '林深时见鹿，海蓝时见鲸，梦醒时见你'
  keywords: 陈卓然的博客
  author: CzrTuringB
  language: zh-CN
  timezone: Asia/Shanghai
  ```

- URL配置：

  ```yaml
  #域名信息
  url: http://czr.czrturingb.top
  permalink: :year/:month/:day/:title/
  permalink_defaults:
  pretty_urls:
    trailing_index: true
    trailing_html: true
  ```

- 相关路径配置：

  ```yaml
  source_dir: source        # 存放你的文章、图片等源文件的目录
  public_dir: public        # Hexo 生成的静态网页最终输出目录
  tag_dir: tags             # 标签页面生成路径
  archive_dir: archives     # 归档页面生成路径
  category_dir: categories  # 分类页面生成路径
  code_dir: downloads/code  # 用于存放下载的代码文件路径
  i18n_dir: :lang           # 国际化语言文件目录
  skip_render:              # 列出不需要 Hexo 渲染的文件或目录
  ```

- 写作配置：

  ```yaml
  new_post_name: :title.md      # 新建文章时文件名格式
  default_layout: post          # 默认文章布局
  titlecase: false              # 文章标题是否自动首字母大写
  external_link:
    enable: true                # 是否对外部链接加 nofollow
    field: site
    exclude: ''                 # 排除哪些站点不加 nofollow
  filename_case: 0              # 文件名大小写，0=原样，1=小写，2=大写
  render_drafts: false          # 是否渲染草稿
  post_asset_folder: false      # 是否为文章生成独立资源文件夹
  relative_link: false          # 是否使用相对链接
  future: true                  # 是否渲染未来日期的文章
  ```

- 代码高亮配置：

  ```yaml
  syntax_highlighter: highlight.js  # 代码高亮方式
  highlight:
    line_number: true      # 是否显示行号
    auto_detect: false     # 是否自动识别语言
    tab_replace: ''
    wrap: true             # 是否自动换行
    hljs: false
  prismjs:
    preprocess: true
    line_number: true
    tab_replace: ''
  ```

- 页面配置：

  ```yaml
  index_generator:
    path: ''            # 首页路径
    per_page: 10        # 每页文章数
    order_by: -date     # 按日期排序，-date 表示最新的在前
  
  default_category: uncategorized
  category_map:         # 可将原分类映射到新分类
  tag_map:              # 可将原标签映射到新标签
  
  meta_generator: true  # 是否生成 meta 标签
  
  date_format: YYYY-MM-DD # 文章日期显示格式
  time_format: HH:mm:ss   # 文章时间显示格式
  
  updated_option: 'mtime' # 使用文件修改时间作为更新时间
  
  per_page: 10            # 其他页面显示文章数
  pagination_dir: page    # 分页目录，例如 /page/2
  ```

- 头文件配置：

  ```yaml
  include:   # 指定要渲染的额外文件
  exclude:   # 指定不渲染的文件
  ignore:    # Hexo 完全忽略这些文件
  ```

- 主题配置：

  ```yaml
  theme: butterfly
  ```

- 部署信息配置：

  ```yaml
  # 部署信息
  deploy:
    type: 
    repo:
    branch:
  ```

### 第二章	Butterfly配置

#### 第一节	搜索栏

- 安装搜索功能依赖：

  ```shell
  #搜索功能
  npm install hexo-generator-search --save
  ```

- 配置搜索：

  ```yaml
  # 搜索栏
  search:
    use: local_search
    local_search:
      enable: true
      preload: true
      top_n_per_article: -1
      unescape: false
      pagination:
        enable: false
        hitsPerPage: 8
      CDN:
  ```

#### 第二节	冒泡特效

- 在Blog/themes/butterfly/source/js目录下新建chocolate.js文件：

  ```js
  $(function() {
      // 气泡
      function bubble() {
          $('#page-header').circleMagic({
              radius: 10,
              density: .2,
              color: 'rgba(255,255,255,.4)',
              clearOffset: 0.99
          });
      }! function(p) {
          p.fn.circleMagic = function(t) {
              var o, a, n, r, e = !0,
                  i = [],
                  d = p.extend({ color: "rgba(255,0,0,.5)", radius: 10, density: .3, clearOffset: .2 }, t),
                  l = this[0];
  
              function c() { e = !(document.body.scrollTop > a) }
  
              function s() { o = l.clientWidth, a = l.clientHeight, l.height = a "px", n.width = o, n.height = a }
  
              function h() {
                  if (e)
                      for (var t in r.clearRect(0, 0, o, a), i) i[t].draw();
                  requestAnimationFrame(h)
              }
  
              function f() {
                  var t = this;
  
                  function e() { t.pos.x = Math.random() * o, t.pos.y = a 100 * Math.random(), t.alpha = .1 Math.random() * d.clearOffset, t.scale = .1 .3 * Math.random(), t.speed = Math.random(), "random" === d.color ? t.color = "rgba(" Math.floor(255 * Math.random()) ", " Math.floor(0 * Math.random()) ", " Math.floor(0 * Math.random()) ", " Math.random().toPrecision(2) ")" : t.color = d.color }
                  t.pos = {}, e(), this.draw = function() { t.alpha <= 0 && e(), t.pos.y -= t.speed, t.alpha -= 5e-4, r.beginPath(), r.arc(t.pos.x, t.pos.y, t.scale * d.radius, 0, 2 * Math.PI, !1), r.fillStyle = t.color, r.fill(), r.closePath() }
              }! function() {
                  o = l.offsetWidth, a = l.offsetHeight,
                      function() {
                          var t = document.createElement("canvas");
                          t.id = "canvas", t.style.top = 0, t.style.zIndex = 0, t.style.position = "absolute", l.appendChild(t), t.parentElement.style.overflow = "hidden"
                      }(), (n = document.getElementById("canvas")).width = o, n.height = a, r = n.getContext("2d");
                  for (var t = 0; t < o * d.density; t++) {
                      var e = new f;
                      i.push(e)
                  }
                  h()
              }(), window.addEventListener("scroll", c, !1), window.addEventListener("resize", s, !1)
          }
      }(jQuery);
  
      // 调用气泡方法
      bubble();
  })
  ```

- 在inject的bottom中添加：

  ```yaml
  - <script defer src="https://npm.elemecdn.com/jquery@latest/dist/jquery.min.js"></script>
  - <script data-pjax defer src="https://npm.elemecdn.com/tzy-blog/lib/js/theme/chocolate.js"></script>
  ```

  