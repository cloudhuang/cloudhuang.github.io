---
title: "Rails生成Ext Tree"
date: 2008-03-29
categories: ["Ruby on Rails"]
tags: ["Ruby on Rails", "Tree"]
---

> 树形菜单是在开发中经常会遇到的一个功能，RoR的这个设计对于大数据量tree结构是最好的，比如一些常见的查询都可以避免循环或者递归抓取, 而且通过在lft, rgt上设置索引能够达到最好的查询效率。(发布于JavaEye)

在Rails中使用has_one 、has_many 、belongs_to 和 has_and_belongs_to_may 来声明关系型数据库中的一对一，一对多和多对多的关系，但当想以树形的数据结构来表示分类的时候，这些基本的关联功能并不够，Rails在has_XXX关系的基础上，提供了acts as的扩展功能，如`acts_as_list` 、`acts_as_tree` 、 `acts_as_nested_set`。`acts_as_tree`就提供树状的结构来组织记录。(不知道为什么Rails2.0以后会取消掉，需要通过插件的方式来安装)

**`acts_as_nested_set`的官方解释：**
> A Nested Set is similar to a tree from ActsAsTree. However the nested set allows building of the entire hierarchy at once instead of querying each nodes children, and their children. When destroying an object, a before_destroy trigger prunes the rest of the branch of object under the current object.

上面是引用自rubyonrails.org上的对于`acts_as_nested_set`的描述，并提供了一个简单的示例：

**SQL脚本：**

```
create table nested_objects (  
  id int(11) unsigned not null auto_increment,  
  parent_id int(11),  
  lft int(11),  
  rgt int(11),  
  name varchar(32),  
  primary key (id)  
);  
```

**Ruby Model:**
```
class NestedObject < ActiveRecord::Base  
  acts_as_nested_set  
end 
```

**`acts_as_nested_set`提供的方法：**

*   root?() – _是否是根对象_
*   child?() – _是否是子对象(拥有父对象)_
*   unknown?() – _不知道该对象的状态(既不是根对象，也不是子对象)_
*   add_child(child_object) – _为根对象添加一个子对象(如果child_object是一个根对象的话，则添加失败)_
*   children_count() – _根对象的所有子对象的个数_
*   full_set() – _重新找到所有对象_
*   all_children() – _根对象的所有子对象_
*   direct_children() –_根对象的直接子对象_

下面就使用`acts_as_nested_set`来生成一个Ext的Tree。

比如生成如下的树：
```
root
    |_ Child 1
    |  |_ Child 1.1
    |  |_ Child 1.2
    |_ Child 2
       |_ Child 2.1
       |_ Child 2.2
```

先来看一下对上面的树的一个图形化的解释：
![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201019214820.png)

这图还是比较清除的，请理解横线中的1到14这些数字，对应这个树，我们可能会有下面的数据：
![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201019214918.png)

<span style="color: #ff0000;">这个也就是SQL脚本中的的lft和rgt的解释</span>。

**Steps:**

1. 创建Rails工程：
```
rails ExtTree
```

2. 安装act_as_nested_set:
```
ruby script/plugin install acts_as_nested_set
```

3. 下载ext，解压下载后的压缩包并拷贝到ExtTree工程的public目录(public/ext)

4. 创建模型对象：
```
ruby script/generate resource Category parent_id:integer lft:integer rgt:integer text:string
```

5. 给模型对象Category加入acts_as_nested_set:
```
class Category < ActiveRecord::Base  
  acts_as_nested_set  
end  
```

6. 下面在CategoriesController中加入index方法，让它来转到index.html页面，并且为EXT TREE来生成JSON数据：

```
class CategoriesController < ApplicationController  
  def index(id = params[:node])  
    respond_to do |format|  
      format.html # render static index.html.erb  
      format.json { render :json => Category.find_children(id) }  
    end  
  end  
end  
```

index方法有一个参数id,用来接收一个树的节点的id，我们就可以通过一个id来查找该节点的子节点。

7. 实现CategoriesController中的find_children方法：
```
#首先先得到树的根节点，再根据传过来的id找到根的子节点  
def self.find_children(start_id = nil)  
    start_id.to_i == 0 ? root_nodes : find(start_id).direct_children  
end  

#如果parent_id为空，则为树的根节点  
def self.root_nodes  
    find(:all, :conditions => 'parent_id IS NULL')  
end  
```

到这里，已经实现了基本的树形结构，但却还有一个问题，如果是树叶节点，既没有子节点的节点，图标应该显示为"-" ，不应该再能够伸展了，Ext Tree中提供的示例中给出的JSON数据中有一个leaf的属性，如果为true，则为树叶节点，如果为false，则为树枝节点，所以，我们还需要让我们生成的JSON数据用来leaf来标识树枝节点与树叶节点，在Category.rb中添加如下代码：

```
def leaf  
    unknown? || children_count == 0  
end  
  
def to_json_with_leaf(options = {})  
    self.to_json_without_leaf(options.merge(:methods => :leaf))  
end  
  
alias_method_chain :to_json, :leaf 
```

对于alias_method_chain，需要先说一下Ruby中的alias_method方法，在Ruby中有这样的用法：

```
alias_method :old_method_name :new_method_name
```

它同alias很类似，但只能用法方法。

在Ruby中，可以使用方法链的手段来实现mix-in，如果想要用new_method来override old_method方法，就可以这样使用：
```
alias_method :old_method_name :new_method_name  
alias_method :new_method_name :old_method_name  
```

而在Rails中，提供了一个更强大的方法：alias_method_chain。

下面是index.html.erb文件：
```
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"  
        "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">  
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en" lang="en">  
<head>  
    <meta http-equiv="content-type" content="text/html;charset=UTF-8"/>  
    <title>Rails Ext Tree</title>  
    <%= stylesheet_link_tag "../ext/resources/css/ext-all.css" %>  
    <%= javascript_include_tag :defaults %>  
    <%= javascript_include_tag "../ext/adapter/prototype/ext-prototype-adapter.js" %>  
    <%= javascript_include_tag "../ext/ext-all.js" %>  
</head>  
<body>  
<div id="category-tree" style="padding:20px"></div>  
<% javascript_tag do -%>  
    Ext.onReady(function(){       
        root = new Ext.tree.AsyncTreeNode({  
        text: 'Invisible Root',  
        id:'0'  
    });  
     
    new Ext.tree.TreePanel({  
        loader: new Ext.tree.TreeLoader({  
            url:'/categories',  
            requestMethod:'GET',  
            baseParams:{format:'json'}  
        }),  
        renderTo:'category-tree',  
        root: root,  
        rootVisible:false  
    });  
      
    root.expand();  
    });  
<% end -%>  
</body>  
</html>  
```

添加测试数据：
```
root = Category.create(:text => 'Root')  
  
root.add_child(c1 = Category.create(:text => 'Child 1'))  
c1.add_child(Category.create(:text => 'Child 1.1'))  
c1.add_child(Category.create(:text => 'Child 1.2'))  
  
root.add_child(c2 = Category.create(:text => 'Child 2'))  
c2.add_child(c21 = Category.create(:text => 'Child 2.1'))  
c2.add_child(c21 = Category.create(:text => 'Child 2.2'))
```

最后的显示效果：
![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201019215300.png)