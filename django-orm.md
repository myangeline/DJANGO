# django orm

## 1. lookup
在django的orm中，如果要查找一个字段，可能需要用到lookup，比如比较时间，模糊查询等等

    >>> Blog.objects.filter(title__contains='xxx') # 模糊查询
    >>> Blog.objects.filter(create_date__gte='xxx', create_date__lte='xxx') # 大于等于的比较
    >>> Blog.objects.filter(id__in = [1, 2, 3]) # 在某个范围内
更多操作查看官网文档:
> https://docs.djangoproject.com/en/1.8/ref/models/querysets/

如果有外键关联的字段，直接使用会有错误，需要  `field__field__lookup`，例如：
    
    >Blog.objects.filter(author_id__id__contains = 'xxx') # Blog的一个字段author_id是一个外键，再模糊查询这个外键
    