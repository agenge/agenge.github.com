---
layout: post
title: "octopress个性化定制"
date: 2013-10-10 13:39
comments: true
categories: 
- Blog

---

## 设置导航栏
编辑octopress\source\_includes\custom\navigation.html, 并加入以下内容：

```
<ul class="main-navigation">
  <li><a href="{{ root_url }}/">首页</a></li>
  <li><a href="{{ root_url }}/blog/archives">归档</a></li>
    <li><a href="{{ root_url }}/about">关于我</a></li>
</ul>
```

## 添加 "关于我"导航页

```
$ rake new_page['about']
mkdir -p source/about
Creating new page: source/about/index.markdown
```

## 侧边栏文章分类  
创建文件：octopress\plugins\category_list_tab.rb，内容如下：

```
module Jekyll 
  class CategoryListTag < Liquid::Tag 
    def render(context) 
      html = "" 
      categories = context.registers[:site].categories.keys 
      categories.sort.each do |category| 
        posts_in_category = context.registers[:site].categories[category].size 
        category_dir = context.registers[:site].config['category_dir'] 
        category_url = File.join(category_dir, category.gsub(/_|\P{Word}/, '-').gsub(/-{2,}/, '-').downcase) 
        html << "<li class='category'><a href='/#{category_url}/'>#{category} (#{posts_in_category})</a></li>\n" 
      end 
      html 
    end 
  end 
end

Liquid::Template.register_tag('category_list', Jekyll::CategoryListTag)
```

新建文件 octopress\source\_includes\custom\category_list.html,并添加以下内容(由于偶本地生成一直报错，请注意在使用的时候去掉反斜扛\)：

```
<section> 
  <h1>文章分类</h1> 
  <ul id="categories"> 
    {\% category_list \%} 
  </ul> 
</section>
```

修改 _config.yml文件 ：

```
default_asides: [asides/recent_posts.html, asides/category_list.html, asides/github.html, asides/delicious.html, asides/pinboard.html, asides/googleplus.html]
```

修改octopress\source\_includes\asides\recent_posts.html

```
  <h1>Recent Posts</h1>
```
修改为

```
<h1>最新文章</h1>
```

## 显示 文章以摘要形式展示
在markdown文件需要显示的地址添加 <!-- more -->，在发布文章后就会显示 Read On链接。
如果想将Read On显示为：阅读全文，请修改_config.yml,将

```
excerpt_link: "Read on &rarr;" 
```

修改为

```
excerpt_link: "阅读全文 &rarr;" 
```

## 添加版权声明
在 source\_includes\post 目录下创建一个文件：license.html，内容如下：

```
{% if site.post_license  %}

<DIV style="font-size:12px;BORDER-BOTTOM: #bbbbbb 1px solid; BORDER-LEFT: #bbbbbb 1px solid; BACKGROUND: #f6f6f6; HEIGHT: 120px; BORDER-TOP: #bbbbbb 1px solid; BORDER-RIGHT: #bbbbbb 1px solid" class=oec2003right>
<DIV style="MARGIN-TOP: 10px; FLOAT: left; MARGIN-LEFT: 5px; MARGIN-RIGHT: 10px"></DIV>
<DIV style="LINE-HEIGHT: 200%; MARGIN-TOP: 10px; COLOR: #000000">
作者： <A href="http://agenge.com">agenge</A> <BR>
出处： <A href="http://agenge.com">http://agenge.com</A> 
<BR>本博客所有的原创文章，作者皆保留版权。转载必须包含本声明，保持本文完整，并以超链接形式注明作者
<a href=mailto:huanggeng.8552@gmail.com> agenge </a>。 </DIV></DIV>

{% endif %}
```

在sass\custom_styles.scss  添加样式，内容如下：

```
.oec2003right
{
    background: #C3D9FF;
    height:120px;
    border:1px solid #BBBBBB;
}

.oec2003right a:link 
{
    color: #0057b6;
    text-decoration: none;
}
.oec2003right a:visited 
{
    color: #0057b6;
    text-decoration: none;
}
.oec2003right a:active,a:hover
{
    color: #0057b6;
    text-decoration: underline;
}
```

修改 source\_layouts\post.html, 在第14行(由于偶本地生成一直报错，请注意在使用的时候去掉反斜扛\)

```
	{% include post\/categories.html %}
```
这行之后, 添加(由于偶本地生成一直报错，请注意在使用的时候去掉反斜扛\)

```
	{% include post\/license.html %}
```
在_config.yml最后一行添加以下内容，意思为控制是否显示页面的版权信息,True为显示

```
post_license: true
```

效果如下：

<DIV style="font-size:12px;BORDER-BOTTOM: #bbbbbb 1px solid; BORDER-LEFT: #bbbbbb 1px solid; BACKGROUND: #f6f6f6; HEIGHT: 120px; BORDER-TOP: #bbbbbb 1px solid; BORDER-RIGHT: #bbbbbb 1px solid" class=oec2003right>
<DIV style="MARGIN-TOP: 10px; FLOAT: left; MARGIN-LEFT: 5px; MARGIN-RIGHT: 10px"></DIV>
<DIV style="LINE-HEIGHT: 200%; MARGIN-TOP: 10px; COLOR: #000000">
作者： <A href="http://agenge.com">agenge</A> <BR>
出处： <A href="http://agenge.com">http://agenge.com</A> 
<BR>本博客所有的原创文章，作者皆保留版权。转载必须包含本声明，保持本文完整，并以超链接形式注明作者
<a href=mailto:huanggeng.8552@gmail.com> agenge </a></DIV></DIV>

## 域名设置
域名服务商设置

主机名    记录类型	地址
空		A记录		207.97.227.245
www		CNAME		agenge.github.com

```
echo "agenge.com" >> source/CNAME
```
---
以上主要是 agenge.com 这个Blog的常见设置，如果你觉得设置很麻烦，请直接git源代码吧: github.com/agenge.github.com
