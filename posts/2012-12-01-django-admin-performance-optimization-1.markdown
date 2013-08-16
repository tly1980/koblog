.. layout: post
.. title: "Django Admin 的性能优化一"
.. date: 2012-12-01 15:35
.. comments: true
.. categories: [技术, django]
.. keywords: django, 性能优化, admin
.. description: 如何帮 Django Admin 做性能优化


> Django Admin 让程序猿不写太多代码就生产一功能基本齐备的后端，确实是很好很强大的。但很好很强大的工具也通常比较笨，会有多余操作，导致性能不大好。优化本来应针对业务，尽量不要去碰后台管理的优化，除非真的很闲。
> 然而，Admin 真的是慢得让开发人员也无法忍受，一个页面产生500条语句，耗时十几秒，所以忍不住要去改改。在某些项目，Admin 页面就是产品的正式后端管理工具，那部分是有企业客户用的。如果想顺利收到钱，不想被xxoo的话，也有优化 Admin 的需要。

问题：List 页面当使用到外键产生过多的查询
-----------------------------------

select_related 可以设置为 True。 因为 Django ORM 默认是 Lazy Load 的，通常外键的属性不会在产生 List 结果的 QuerySet 里头被装载，而要等到你调用其属性了才会装载。
譬如有一个 Model 叫 Blog，有一个外键 Author。

```python
class Author(models.Model):
    first_name = models.Charfield(max_length=64)
    last_name = models.Charfield(max_length=64)
    email = models.Charfield(max_length=255)
    ...
    ...

class Blog(models.Model):
    title = models.Charfield(max_length=1024)
    content = models.TextField()
    author = models.ForeignKey(Author)

    def author_name():
        return '%s %s' % (author.first_name, author.last_name)


```

然后你希望在 Admin list 里头见到 Author 通常这么做：

```python

class BlogAdmin(admin.ModelAdmin):
    list_display = ('id', 'title', 'author_name')
```

这个时候，如果调用 Admin 的 Blog 的 list page 通常产生 100 条以上查询语句。因为 author_name 方法会引用到外键`对象 author ，而 Admin list 分页默认是 100 条记录为一个分页。
100 条查询语句，现在的机器那么牛，实在不能算什么。而如果你有 n 个外键需要被显示，就会产 n * 100 条语句，而且他们都是串行执行的，因为是 Lazy 装载，而在 Template 里头是逐个被显示出来。。。


解决办法就是让 BlogAdmin 在产生 queryset 的时候对外键 Author 产生 JOIN 。
注意，上面的代码里，Blog.author 这个field 因为没设置 null=True，因此会产生 INNER JOIN，反之产生 LEFT OUTER JOIN，因为 author 可为空。

解决：办法有二
------------
1). 修改 BlogAdmin 的 select_related 开关。

```python

class BlogAdmin(admin.ModelAdmin):
    select_related = True
    list_display = ('id', 'title', 'author_name')
    ...
```

这个办法的好处就是简单，缺点是不够灵活。因为这个会把 Blog 所有的外键都 JOIN 进来，不管他们是否在 list_display 里头设置了。

2). 重载 Admin 的 queryset 方法。
```python

class BlogAdmin(admin.ModelAdmin):
    list_display = ('id', 'title', 'author_name')
    
    def queryset(self, request):
        qs = super(BlogAdmin, self).queryset(request)
        qs = qs.select_related('author')
        return qs

    ...
```

在此你可以显式指定那些外键需要被重载。不仅如此，select_related 还允许继续将外键的外键级联叠加进来。
譬如：

```python
qs = qs.select_related('author', 'author__nationality')
```
上面的例子是不仅连 author 会 join，连 nationality 也会被 join 。
虽然没有做过实验，但是猜测应该是可以无限级联下去。
因为 author 已经是外键了，所以 select_related 中可以将 author 参数去掉，简写成如下：

```python
qs = qs.select_related('author__nationality')
```
