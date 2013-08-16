.. layout: post
.. title: "Django Admin 的性能优化二"
.. date: 2012-12-01 15:35
.. comments: true
.. categories: [技术, django]
.. keywords: django, 性能优化, admin, queryset
.. description: 如何帮 Django Admin 做性能优化


> Django Admin 让程序猿不写太多代码就生产一功能基本齐备的后端，确实是很好很强大的。但很好很强大的工具也通常比较笨，会有多余操作，导致性能不大好。优化本来应针对业务，尽量不要去碰后台管理的优化，除非真的很闲。
> 然而，Admin 真的是慢得让开发人员也无法忍受，一个页面产生500条语句，耗时十几秒，所以忍不住要去改改。在某些项目，Admin 页面就是产品的正式后端管理工具，那部分是有企业客户用的。如果想顺利收到钱，不想被xxoo的话，也有优化 Admin 的需要。

问题：List 页面当外键需要做聚合查询统计的时候
--------------------------------------

请想象你要做一个电子商务的拍卖，有如下这两个类：
一个 Product 可以有多个 Bid。典型的 1-n 关系，Product 是 Bid 的外键。

```python

class Product:
    ...

    def bid_count():
        return self.bids.count()

    def bid_accepted():
        return True if self.bids.filter(accepted=True).count() > 0 else False


class Bid(models.Model):
    accepted = models.Boolean(null=True, default=None)
    product = models.ForeignKey(Product)
    ...
```

很简单很直观吧。好了，然后你有这样一个 admin，问题就来了 。。。
```python
class ProductAdmin(admin.ModelAdmin):
    display_list = ('id', 'bid_count', 'bid_accepted', ... )

```

通过 select_related 搞定了其他的外键，但就是 bid_count 搞不定啊，而 bid_count 跟 bid_accepted 每个就引起一次 sql 查询，一个 Admin 的 list 页面 200 条 SQL 呢。    
山穷水复疑无路，柳暗花明又一村。    
重载 Admin 的 queryset 方法，搞定了 bid_count:

```python

class ProductAdmin(admin.ModelAdmin):
    display_list = ('id', 'bid_count', 'bid_accepted', ... )
    
    def queryset(self, request):
        qs = super(BlogAdmin, self).queryset()
        qs = qs.select_related( ...) # 这里去搞定其他外键
        qs = qs.annotate(bid_count = Count(bids))
        return qs
```

好的这个 annotate 下去，搞定了 100 条 SQL，结果还剩下 100 条呢，bid_accepted 可以搞定么？？    
所以很自然就想到如下的语句

```python
qs = qs.annotate(bid_count=Count(bids, bids__accepted=True))
```

生成的 sql 如下：
```sql
 SELECT *,
     COUNT(`bid`.`id`) AS `bid_count`, 
     COUNT(`bid`.`id`) AS `bid_accepted` FROM 
         `product` LEFT OUTER JOIN `bid` ON (`product`.`id` = `bid`.`product_id`) 
         GROUP BY 
            product.* -- 懒得写了，反正就是 product 的每一项。
```
显然，这并非我们想要的，bid_accepted 也丢失了。

经过一番摸索，我发现了 queryset 还有一个很好很强大的 extra 功能，允许我们嵌入子查询所以将代码改成如下：
```python
qs = qs.extra({
    'bid_count': 'SELECT COUNT(*) FROM bid \
        WHERE product.id = bid.product_id and bid.accepted=1',

    'bid_accepted': 'SELECT COUNT(*) FROM bid\
        WHERE product.id = bid.submission_id and haggler_bid.accepted=1',
})
```

生成 SQL 如下
```sql
SELECT 
    (SELECT COUNT(*) FROM bid WHERE product.id = bid.product_id) AS `bid_count`, 
    (SELECT COUNT(*) FROM bid WHERE product.id = bid.product_id and bid.accepted=1) 
        AS `bid_accepted`, 
    product.*
         FROM product ORDER BY product.id DESC LIMIT 100
```

正是鄙人想要的~ :)。

从 Django Debug Tool 上看到的查询语句数马上降低为 3 条，查询时间也降低到 35ms 内，耗时仅为优化前的 1/10 多一些。  
即使这条查询语句含有子查询，但也总比100多200条查询好多了！


其实这个 extra 完全可以自定制 product 的 model manager 里头，不过项目刚上手，还不是很熟悉，还是先不冒险了。
