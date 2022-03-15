# 创建Zeal可用的Docset文件

## Rust Package Document (Cargo)

使用[cargo-docset](https://github.com/Robzz/cargo-docset)或者[rsdocs-dashing](https://github.com/hobofan/rsdocs-dashing)创建rust的文档docset，并使用"Docset的改进"中的技巧进行美化

## Doxygen生成docset

对于C, C++, C#, PHP, Obj-C, Java, Python文件，可以生成源文件的docset。配置doxygen时，需要使能如下的选项

```doxygen
GENERATE_DOCSET   = YES
// below 3 options are optional
DISABLE_INDEX     = YES 
SEARCHENGINE      = NO
GENERATE_TREEVIEW = NO
```

在doxygen生成文档时，可能需要`docsetutil`这个工具，可以通过包管理器尝试安装

##  Any HTMLs Docset文档创建

1.  创建相关docset的目录，并将现有文档的HTML文件夹拷入

   ```shell
   mkdir -p <docset name>.docset/Contents/Resources/Documents
   cp -rf <source dirs> <docset name>.docset/Contents/Resources/Documents
   ```

2.  在Contents目录创建一个Info.plist的文件，更新内容如下

   ```shell
   cat <<- EOF > <docset name>.docset/Contents/Info.plist
   <!-- create docset Info.plist -->
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
   <plist version="1.0">
   <dict>
   	<!-- ~~~docset name~~~ -->
   	<key>CFBundleIdentifier</key>
   	<string>nginx</string>
   	<!-- ~~~Name show in zeal, Important!!~~~ -->
   	<key>CFBundleName</key>
   	<string>Nginx</string>
   	<!-- ~~~name in dash website? most time same as CFBundleIdentifier~~~ -->
   	<key>DocSetPlatformFamily</key>
   	<string>nginx</string>
   	<!-- ~~~index page~~~ -->
   	<key>dashIndexFilePath</key>
     	<string>nasmdoc0.html</string>
     	<!-- ~~~dash type docset~~~ -->
   	<key>isDashDocset</key>
   	<true/>
   	<!-- ~~~Optional: support TOC~~~ -->
   	<key>DashDocSetFamily</key>
   	<string>dashtoc</string>
   </dict>
   </plist>
   EOF
   ```

3.  创建\<docset name\>.docset/Contents/Resources/docSet.dsidx文件(**该文件是zeal实现页面查找的关键**)；该步骤需要使用脚本语言进行创建

   ```sql
   /* create SQLlite database, which is docSet.dsidx */
   CREATE TABLE searchIndex(id INTEGER PRIMARY KEY, name TEXT, type TEXT, path TEXT);
   
   /* prevent adding duplicate entries to index */
   CREATE UNIQUE INDEX anchor ON searchIndex (name, type, path);
   
   /* scan HTML and add appropriate rows to database by below statements */
   INSERT OR IGNORE INTO searchIndex(name, type, path) VALUES ('name', 'type', 'path');
   /*
    * item meanings:
    *		name: entry name. Ex, if it's class, that's the class name, which the column that Zeal search
    *		type: entry type. Zeal recognize types see next "Zeal supported types"
    * 		path: relative path towards documentation file with Zeal display this entry. can contain anchor(#), also `http://` URL entry
   */
   ```

   不同语言的生成脚本

   - python-better  https://github.com/iamaziz/PyTorch-docset.git
   - python             https://github.com/drbraden/pgdash

   ```python
   #!/usr/bin/env python
   
   import os, re, sqlite3
   from bs4 import BeautifulSoup, NavigableString, Tag 
   
   # create docset sqlite database
   conn = sqlite3.connect('postgresql.docset/Contents/Resources/docSet.dsidx')
   cur = conn.cursor()
   
   try: 
     # remove any search index table
     cur.execute('DROP TABLE searchIndex;')
   except: pass
   
   # create database table
   cur.execute('CREATE TABLE searchIndex(id INTEGER PRIMARY KEY, name TEXT, type TEXT, path TEXT);')
   cur.execute('CREATE UNIQUE INDEX anchor ON searchIndex (name, type, path);')
   
   # set document root dir
   docpath = 'postgresql.docset/Contents/Resources/Documents'
   
   # open index/home page
   page = open(os.path.join(docpath,'bookindex.html')).read()
   soup = BeautifulSoup(page)
   
   # find any hyperlink page, add it
   any = re.compile('.*')
   for tag in soup.find_all('a', {'href':any}):
       name = tag.text.strip()
       if len(name) > 1:
           path = tag.attrs['href'].strip()
           if path != 'index.html':
               cur.execute('INSERT OR IGNORE INTO searchIndex(name, type, path) VALUES (?,?,?)', (name, 'func', path))
               print 'name: %s, path: %s' % (name, path)
   
   # commit to database & close
   conn.commit()
   conn.close()
   ```

   - ruby                 [https://github.com/Kapeli/erlang-docset](https://github.com/Kapeli/erlang-docset/blob/master/src/generate.rb)
   - Node.js           https://github.com/exlee/d3-dash-gen
   - PHP                https://github.com/akirk/dash-phpunit

4.  ***『Optional』*** 为当前docset添加TOC (tabel of contents) [在zeal中在左下角进行显示]。为了支持当前页面的TOC，需要扫描所有的HTML页面，并在需要支持TOC的HTML文件中添加Zeal可识别的Anchor，格式如下

   ```html
   <a name="//apple_ref/cpp/Entry Type/Entry Name" class="dashAnchor"></a>
   <!--
   	Entry type: Zeal Supported Types as below
   	Entry name: name show in Zeal TOC, best <percent escape>
   -->
   ```

   pyton例子：[https://github.com/jkozera/zeal/blob/master/gendocsets/extjs/parse.py](https://github.com/jkozera/zeal/blob/master/gendocsets/extjs/parse.py)

   在这之后，需要在Info.plist中添加新的entry

   ```xml
   <key>DashDocSetFamily</key>
   <string>dashtoc</string>
   ```

## Docset的改进

1.  将docset贡献到Dash

   [https://github.com/Kapeli/Dash-User-Contributions](https://github.com/Kapeli/Dash-User-Contributions#contribute-a-new-docset)  自己用，不需要

2.  添加index页面 (首页)

   ```xml
   <key>dashIndexFilePath</key>
   <string>index.html</string>
   
   <!-- html页面必须是针对Documents的相对路径 -->
   ```

3.  添加图标

   设置icon.png文件在\<docset name\>.docset文件夹中；图表大小必须为16x16或者32x32大小；如果含有两个不同size的icon文件，则16x16(icon.png)，32x32(icon@2x.png)

4. 支持在线文档重定向

   从zeal 3.0开始，可以在某个docset中打开网页中的文档。可以按照如下2种方法进行配置

   - 在Info.plist中设置`DashDocSetFallbackURL = <website address>`，例如: https://docs.python.org/3/library/

   - 在docset的目录中的每个HTML页面中添加注视标签

     ```html
     <html><! -- online page at https://docs.python.org/3/library/intro.html -->
     ```

5. 添加代码交互功能

   从zeal 4.0开始，可以在必要的代码片段处添加"play groud"按钮，用于和用户进行交互。需要在Info.plist文件中添加配置信息`DashDocSetPlayURL = <interactive website>`

6. 添加Javascript的支持

   默认情况下，zeal不执行外部.js文件，如需要支持，在Info.plist文件中添加配置`<key>isJavaScriptEnabled</key><true/>`，然后，需要在zeal中重新添加相应文档的位置，通过Preferences

7. 全文搜索功能

   在Info.plist进行配置

   ```xml
   <key>DashDocSetDefaultFTSEnabled</key><true/>	 <!-- enable full text search -->
   <key>DashDocSetFTSNotSupported</key><true/>  <!-- disable full text search -->
   ```

8. 创建本地docset订阅服务(feed)

   可以为某一个docset创建一个XML文件，使用本地服务器的方式(nigix)创建本地订阅服务，从而手动获得文档文件的更新。feed文件的格式

   ```xml
   <entry>   <!-- each file only one entry -->
   <version>0.10.26</version>			<!-- version info -->
   <!-- candidate urls for download -->
   <url>http://newyork.kapeli.com/feeds/NodeJS.tgz</url>
   <url>http://sanfrancisco.kapeli.com/feeds/NodeJS.tgz</url>
   <url>http://london.kapeli.com/feeds/NodeJS.tgz</url>
   <url>http://tokyo.kapeli.com/feeds/NodeJS.tgz</url>
   </entry>
   ```

   对于tgz的创建，使用如下命令`tar --exclude='.DS_Store' -cvzf <docset name>.tgz <docset name>.docset`

9. 分享docset订阅服务(feed)

   可以在网络进行订阅服务的发布，需要创建一个特定的URL格式，如下

   ```web-idl
   dash-feed://<URL encoded feed URL>
   ```

## Zeal Supported Types

Annotation	Attribute	Binding	Builtin	Callback	Category	Class	Command	Component

Constant	Constructor	Define	Delegate	Diagram	Directive	Element	Entry	Enum

Environment	Error	Event	Exception	Extension	Field	File	Filter	Framework	Function

Global	Guide	Hook	Instance	Instruction	Interface	Keyword	Library	Literal	Macro

Method	Mixin	Modifier	Module	Namespace	Notation	Object	Operator	Option	Package

Parameter	Plugin	Procedure	Property	Protocol	Provider	Provisioner	Query	Record

Resource	Sample	Section	Service	Setting	Shortcut	Statement	Struct	Style	Subroutine

Tag	Test	Trait	Type	Union	Value	Variable	Word

## 参考

1. [Kapeli Docset Generation Guide](https://kapeli.com/docsets#setUpFolderStructure)