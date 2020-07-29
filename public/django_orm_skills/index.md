# 一次django内存异常排查




## 起因



`Django` 作为 `Python`著名的`Web`框架，相信很多人都在用，自己工作中也有项目项目在用，而在最近几天的使用中发现，部署`Django`程序的服务器出现了内存问题，现象就是运行一段时间之后，内存占用非常高，最终会把服务器的内存耗尽，对于`Python`项目出现内存问题，自己之前处理过一次，所以并没有第一次解决时的慌张，自己之前把解决方法也整理了博客：https://www.cnblogs.com/zhaof/p/10031945.html 



但是事情似乎并没有我想的那么简单，自己尝试用之前的的方法`tracemalloc`库进行问题的排查，但是问题来了实际的项目中有快一百多个接口，怎么排查？难道一个一个接口进行测试排查，但是时间又比较紧急，可能又来不及了。对比上次自己解决是因为上次的项目比较简单，相对来说定位问题比较容易，那么这次怎么处理呢？





## 处理过程



一般`Python`项目其实是很少出现内存问题的，一般都是自己代码写的有问题导致的，而对于这次出现的问题，自己的排查思路（对于web 接口类型的项目）：

1. 先排查调用比较频繁的接口
2. 然后排查数据汇总接口（查询比较复杂）
3. 如果上述还没有查出来，再排查剩余的接口



在这次的问题排查中，自己大致也是按照这个思路进行的，在对调用频繁的接口进行排查时，并没有发现内存的异常，而出现内存的问题则是在数据汇总的相关接口上。

其实这种接口对于初级开发可能是容易出问题的地方，首先这种接口查询的数据相对其他接口会比较复杂，如果编码基础又不是特别好，可能就会在这些接口上出现bug. 

而在这次的排查中，最终确定是在一个汇总数据的接口上，定位到问题处在了`Django ORM` 使用不当导致的。自己通过一个简单代码实例来说明：



```python
class Student(models.Model):
    name = models.CharField(max_length=20)
    name2 = models.CharField(max_length=20)
    name3 = models.CharField(max_length=20)
    name4 = models.CharField(max_length=20)
    name5 = models.CharField(max_length=20)
    name6 = models.CharField(max_length=20)
    name7 = models.CharField(max_length=20)
    name8 = models.CharField(max_length=20)
    name9 = models.CharField(max_length=20)
    name10 = models.CharField(max_length=20)
    name11 = models.CharField(max_length=20)
    name12 = models.CharField(max_length=20)
    name13 = models.CharField(max_length=20)
    name14 = models.CharField(max_length=20)
    name15 = models.CharField(max_length=20)
    age = models.IntegerField(default=0)
```

正常情况，我们的表字段会比较多，这里就通过多个name来模拟，出现题的代码就出在关于这个表的接口上：

```python
def index(request):
    studets = Student.objects.filter(age__gt=20)
    if studets:
        pass
    return HttpResponse("test memory")
```

为了让内存问题容易复现，我通过脚本向Student中插入了20000条数据，当然这里数据越多，问题越明显

通过一个测试脚本并发请求这个接口，观察内存情况，你会发现，内存会出现瞬间上涨的情况，并且如果你的数据越多，请求越多，你的内存可能会在一段时间居高不下，并且逐渐上涨。问题出在哪里了？



其实很简单，问题出在了代码中的if 判断那里，我们通过filter 查询返回的是QuerySet 类型的数据，而我们过滤之后的数据可能会存在非常多的时候，这个时候我们通过if 直接判断，自己的理解这个地方会将整个QuerySet加载到内存中，从而出现内存占用过高的问题，而如果并且这个时候这个接口的响应速度也是非常会变慢，而这个QuerySet 中的数据越多，内存占用越明显。



在`Django`的文档中其实做了说明

> - `exists`()[¶](https://docs.djangoproject.com/en/3.0/ref/models/querysets/#django.db.models.query.QuerySet.exists)
>
>   
>
> Returns `True` if the [`QuerySet`](https://docs.djangoproject.com/en/3.0/ref/models/querysets/#django.db.models.query.QuerySet) contains any results, and `False` if not. This tries to perform the query in the simplest and fastest way possible, but it *does* execute nearly the same query as a normal [`QuerySet`](https://docs.djangoproject.com/en/3.0/ref/models/querysets/#django.db.models.query.QuerySet) query.
>
> [`exists()`](https://docs.djangoproject.com/en/3.0/ref/models/querysets/#django.db.models.query.QuerySet.exists) is useful for searches relating to both object membership in a [`QuerySet`](https://docs.djangoproject.com/en/3.0/ref/models/querysets/#django.db.models.query.QuerySet) and to the existence of any objects in a [`QuerySet`](https://docs.djangoproject.com/en/3.0/ref/models/querysets/#django.db.models.query.QuerySet), particularly in the context of a large [`QuerySet`](https://docs.djangoproject.com/en/3.0/ref/models/querysets/#django.db.models.query.QuerySet).
>
> The most efficient method of finding whether a model with a unique field (e.g. `primary_key`) is a member of a [`QuerySet`](https://docs.djangoproject.com/en/3.0/ref/models/querysets/#django.db.models.query.QuerySet) is:
>
> ```
> entry = Entry.objects.get(pk=123)
> if some_queryset.filter(pk=entry.pk).exists():
>     print("Entry contained in queryset")
> ```
>
> Which will be faster than the following which requires evaluating and iterating through the entire queryset:
>
> ```
> if entry in some_queryset:
>    print("Entry contained in QuerySet")
> ```
>
> And to find whether a queryset contains any items:
>
> ```
> if some_queryset.exists():
>     print("There is at least one object in some_queryset")
> ```
>
> Which will be faster than:
>
> ```
> if some_queryset:
>     print("There is at least one object in some_queryset")
> ```
>
> … but not by a large degree (hence needing a large queryset for efficiency gains).
>
> Additionally, if a `some_queryset` has not yet been evaluated, but you know that it will be at some point, then using `some_queryset.exists()` will do more overall work (one query for the existence check plus an extra one to later retrieve the results) than using `bool(some_queryset)`, which retrieves the results and then checks if any were returned.



所以对于我们的代码我们只需要把if 判断地方改成`if not studets.exists()` 就可以解决问题。

这是一个很小的知识点，但是如果使用不对，可能就会造成非常严重的内存问题。



## 总结

- 除了单元测试，还需要做大数据量测试，这次的问题如果在测试的时候做过一定数据量的测试，可能很早就能及时发现问题
- 对于基础的库的使用要更加熟悉
- 排查问题的思路要明确，不然可能会无从下手



### 延伸阅读

- https://docs.djangoproject.com/en/3.0/ref/models/querysets/
- https://www.cnblogs.com/zhaof/p/10031945.html


