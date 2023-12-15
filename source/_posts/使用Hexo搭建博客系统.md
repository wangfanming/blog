# 使用Hexo搭建博客系统

## Hexo简介

Hexo是一款基于Node.js的静态博客框架，依赖少易于安装使用，可以方便的生成静态网页托管在GitHub和Coding上，是搭建博客的首选框架。大家可以进入[Hexo官网](https://hexo.io/zh-cn/index.html)进行详细查看。

## Hexo搭建步骤

- 安装Git和Node.js
- 安装Hexo
- 配置Github Actions

> Git和Node.js基础环境正常安装即可，只需要对Github做一下密钥登录，相关操作在网上有大量博客，不再赘述。

## 安装Hexo

1. 安装hexo-cli脚手架

   ```shell
   npm install -g hexo-cli	
   ```

2. 创建博客根目录

   ```shell
   mkdir blog	
   ```

3. 初始化博客仓库

   ```shell
   hexo init blog	
   ```

   - 新建完成后，指定文件夹目录下有：

     1. `node_modules` : 依赖包

     2. `public` ：存放生成的页面

     3. `scaffolds` ：生成文章的一些模板

     4. `source` ：用来存放你的文章，MarkDown格式的文章，在`hexo g`生成时，会被自动构建出网页版，存放在public目录下

     5. `themes` ：主题目录

     6. `_config.yml` : **博客的配置文件**，具体配置参考官方，以下是我的配置

        ```shell
        # Hexo Configuration
        ## Docs: https://hexo.io/docs/configuration.html
        ## Source: https://github.com/hexojs/hexo/
        
        # Site
        title: Duncan's Blog
        subtitle: '记录学习 分享生活 品味人生'
        description: '吾生也有涯，而知也无涯，以有涯随无涯，殆己！已而为知者，殆而已矣。'
        keywords: 
        author: Wang Fanming
        language: zh-CN
        timezone: 'Asia/Shanghai'
        
        # URL
        ## Set your site url here. For example, if you use GitHub Page, set url as 'https://username.github.io/project'
        url: https://www.wfanming.top
        root: /
        permalink: :year/:month/:day/:title/
        permalink_defaults:
        pretty_urls:
          trailing_index: true # Set to false to remove trailing 'index.html' from permalinks
          trailing_html: true # Set to false to remove trailing '.html' from permalinks
        
        # Directory
        source_dir: source
        public_dir: public
        tag_dir: tags
        archive_dir: archives
        category_dir: categories
        code_dir: downloads/code
        i18n_dir: :lang
        skip_render:
        
        # Writing
        new_post_name: :title.md # File name of new posts
        default_layout: post
        titlecase: false # Transform title into titlecase
        external_link:
          enable: true # Open external links in new tab
          field: site # Apply to the whole site
          exclude: ''
        filename_case: 0
        render_drafts: false
        post_asset_folder: false
        relative_link: false
        future: true
        syntax_highlighter: highlight.js
        highlight:
          line_number: false
          auto_detect: false
          tab_replace: ''
          wrap: true
          hljs: false
        prismjs:
          preprocess: false
          line_number: true
          tab_replace: ''
        
        # Home page setting
        # path: Root path for your blogs index page. (default = '')
        # per_page: Posts displayed per page. (0 = disable pagination)
        # order_by: Posts order. (Order by date descending by default)
        index_generator:
          path: ''
          per_page: 10
          order_by: -date
        
        # Category & Tag
        default_category: uncategorized
        category_map:
        tag_map:
        
        # Metadata elements
        ## https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta
        meta_generator: true
        
        # Date / Time format
        ## Hexo uses Moment.js to parse and display date
        ## You can customize the date format as defined in
        ## http://momentjs.com/docs/#/displaying/format/
        date_format: YYYY-MM-DD
        time_format: HH:mm:ss
        ## updated_option supports 'mtime', 'date', 'empty'
        updated_option: 'mtime'
        
        # Pagination
        ## Set per_page to 0 to disable pagination
        per_page: 10
        pagination_dir: page
        
        # Include / Exclude file(s)
        ## include:/exclude: options only apply to the 'source/' folder
        include:
        exclude:
        ignore:
        
        # Extensions
        ## Plugins: https://hexo.io/plugins/
        ## Themes: https://hexo.io/themes/
        theme: next
        
        # Deployment
        ## Docs: https://hexo.io/docs/one-command-deployment
        deploy:
          type: 'git'
          repo: git@github.com:wangfanming/wangfanming.github.io.git
          branch: main
        
        symbols_count_time:
         #文章内是否显示
          symbols: true
          time: true
         # 网页底部是否显示
          total_symbols: true
          total_time: true
        
        # 通用站点地图
        sitemap:
          path: sitemap.xml
        # 百度站点地图
        baidusitemap:
          path: baidusitemap.xml
        
        
        ```

        

4. 安装相关环境依赖

   ```shell
   cd blog/
   npm install
   ```

5. 启动模板，检查初始化是否正常

   ```shell
   hexo s
   ```

   - 通过[http://localhost:4000](http://localhost:4000/)访问

6. 安装配置[Next主题](https://theme-next.iissnan.com/) ，其他主题配置一样的方法

   - 安装

   ```shell
   #进入博客根目录
   cd blog/
   #下载自己喜欢的主题到Hexo主题目录 `themes/`
   git clone https://github.com/theme-next/hexo-theme-next themes/next
   
   ```

   - 在博客配置文件_config.yml中启用Next主题

   ```
   ## Themes: https://hexo.io/themes/
   theme: next   #主题改为next
   ```

   - 对Next主题的相关配置参考官网即可

7. 部署至本地测试一下，正常输出如下：

   ![hexo博客预览](/images/hexo博客预览.png)

8. 上传博客

   1. 将书写好的MarkDown博客移至source/_posts

   2. 构建网页，并部署

      ```shell
      hexo clean
      hexo g
      hexo d
      ```

      - **部署至Github前，必须保证远程仓库密钥配置成功**

## 配置Github Actions

> 上述博客发布过程，每次必须编译然后进行上传，流程繁琐，且存在本地Node.js升级后与Hexo不匹配的问题，因此可以考虑使用Github Actions的持续集成能力来完成这个自动化过程。

