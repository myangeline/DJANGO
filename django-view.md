# django view

## 1. view function
在django中,view的视图函数很容易创建,作用也很明确的,就是为了处理请求的,对于请求响应前,可以在view中做各种处理,大多数情况下,直接在views.py文件中
创建一个函数就可以,函数的第一个参数一般是request,然后在url中配置好即可.对于后面请求都会映射到这个函数上

    from django.http import HttpResponse
    import datetime
    
    def current_datetime(request):
        now = datetime.datetime.now()
        html = "<html><body>It is now %s.</body></html>" % now
        return HttpResponse(html)

上面就是一个简单的view, 配置下url:

    from django.conf.urls import url

    from . import views
    
    urlpatterns = [
        url(r'^now$', views.current_datetime),
    ]

这样配置好后访问 `http://localhost:8000/now` 就会的到响应啦...

这里的`url`是按照`django 1.9` 设置的,大概从1.8版本以后才可以这样设置,较低的版本设置方式不同,查文档就好了
 
## 2. view decorators
`django`本身提供了一些很有用的视图装饰器,我们在写代码的时候可以用

### 1. require_http_methods
这个装饰器用来限制视图可以被请求的方式,在没有设置的请求方式不可以使用

    from django.views.decorators.http import require_http_methods

    @require_http_methods(["GET", "POST"])
    def my_view(request):
        # I can assume now that only GET or POST requests make it this far
        # ...
        pass
注意: 设置的请求方式需要大写...

从者改革装饰器衍生出来的其他几个装饰器:

* require_GET()
Decorator to require that a view only accepts the GET method.

* require_POST()
Decorator to require that a view only accepts the POST method.

* require_safe()
Decorator to require that a view only accepts the GET and HEAD methods. 
These methods are commonly considered “safe” because they should not have the significance of taking an action 
other than retrieving the requested resource.

其他装饰器先不说了,因为不是很了解,django视图装饰器大部分都是在 `django.views.decorators.*` 这个路径下,所以可以看看源码就知道了

## 2. view class
django内置了几个 view class, 主要集中在 `django.views.generic.*` 这个目录下

其中主要的 `View` 类在 `django.views.generic.base.py`这个模块下,其他的view都是从这里继承过去再加几个`mixin`

细看`View`类中主要的方法: `as_view()`, `dispatch()`

### 1. dispatch()

    def dispatch(self, request, *args, **kwargs):
        if request.method.lower() in self.http_method_names:
            handler = getattr(self, request.method.lower(), self.http_method_not_allowed)
        else:
            handler = self.http_method_not_allowed
        return handler(request, *args, **kwargs)
        
`dispatch()` 这个方法作用其实很简单,就是根据`http`请求方法去调用类中的实现的和请求方法同名的方法.所以这个方法的调用是在实现视图类的方法调用前,但是在
视图类的初始化后.

对于这个方法,如果我们继承这个类之后,我们可以在子类中重写,这样就可以在这个方法中进行一些预处理,之后在视图方法中可以使用到预处理后的结果,
之后的子类可以继续继承这个重写了`dispatch()`方法的类,子类中就可以不用写啦,统一处理还是挺好的

这个方法是在下面 `as_view()` 中调用的,具体看下面的代码
    
### 2. as_view()
    
    @classonlymethod
    def as_view(cls, **initkwargs):

        ...

        def view(request, *args, **kwargs):
            self = cls(**initkwargs)
            if hasattr(self, 'get') and not hasattr(self, 'head'):
                self.head = self.get
            self.request = request
            self.args = args
            self.kwargs = kwargs
            return self.dispatch(request, *args, **kwargs)
        view.view_class = cls
        view.view_initkwargs = initkwargs
        ...
        return view

先贴个源码,其实很简单,我们主要看 `view` 这个方法中的代码,其实在使用类名调用这个`as_view` 方法的时候,得到的就是一个 `view` 方法,
之后再由跟上一级的调用,我猜应该是`wsgi`那边调用的,因为这是一个请求响应的入口,那里还没有仔细看...

最神奇的一个地方是 `dispatch` 这个方法中的 `self` 表示的其实并不是 `dispatch` 所在类的实例(也可以说是,因为是他的子类嘛,总是有关系的.)
而是调用`as_view`的类的实例,原因就是这样一句`self = cls(**initkwargs)`, 而这个`cls`是`as_view`的参数咯,所以就是谁调就是谁的!

`classonlymethod` 这个装饰器的作用从名字应该就可以看得出,就是只能让类调用,不能用实例调用,大概就是这样的咯

最基本的就是这些了,其他的都是一些 `mixin` 的类,配合这个基本的 `View` 组合生成一些其他的类,比如有: `TemplateView`, `RedirectView`等...

### 3. csrf_exempt
这个装饰器在view class中不好用,但是当我们需要的时候怎么办...比如下面的代码:

    class BaseView(TemplateView):

        def post(self, request, *args, **kwargs):
            pass

这个`post`方法开启了`csrf`验证,一般django都是开启全局的,而且那个 middleware的执行时间很早,早到还没到 `dispatch` 调用就已经执行了,所以在
`post`方法上加装饰器没什么用了,有效的办法就是在 `url` 配置的地方使用装饰器,例如:
    
    urlpatterns = [
        url(r'^now$', csrf_exempt(BaseView.as_view()))
    ]

这样是有效的,不过看起来还是怪怪的,没有语法糖的装饰器差评!那么,就需要换一个方式了, 用一个类装饰器就好嘛:
    
    def as_view(exempt=False):
        """
        这个装饰器的作用是去除post之类请求的 csrf_token 的验证
        :param exempt:
        :return:
        """
        def __inner(cls):
            if exempt:
                return csrf_exempt(cls.as_view())
            else:
                return cls.as_view()
        return __inner

在视图类上面这样用:
    
    @as_view(exempt=True)
    class BaseView(TemplateView):

        def post(self, request, *args, **kwargs):
            pass
带了一个参数,其实跟方法的装饰器一个样....不过`url`的配置需要修改,直接使用类名,不要再调用 `as_view` 方法了...
    
    urlpatterns = [
        url(r'^now$', BaseView)
    ]
    

`django`的东西还有很多,不过核心的东西还是那些,不管大小框架,总是有些东西不会变,关键是作者的想法不一样,走的方向就不同了...

`tornado` 和 `webpy` 很像,代码也不多的咯,要学习学习,不过`webpy`的代码说实话不是很pythonic, 尤其是大写的类方法,很不符合pep8规范,
不过还是喜欢,就是因为是我接触过的第一个框架吗?看来还是情怀啊...