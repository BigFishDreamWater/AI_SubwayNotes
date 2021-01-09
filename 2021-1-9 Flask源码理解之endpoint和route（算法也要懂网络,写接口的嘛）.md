2021-1-9 Flask的源码理解之endpoint和route（算法也要懂网络,写接口的嘛）

servelet

如果你自己手写一个响应http请求的服务器，需要

http解析，解析成功，解析失败。

TCP链接复用，

多线程,IO异常处理等等，几千行代码，且需要长期测试。

而servelet则直接封装好了这些代码，提供简单的变成API让开发者使用。

servelet的重定向和转发，重定向是外部转发，浏览器直到转发过程并且记录。

转发web服务器内部将请求发送给另外一个servelet，浏览器并不知道。

以上是java专属的。

而WSGI则是python独有的服务器网关接口协议。全称 web servive gateway interface

Flask则是对WSGI进行更抽象易用封装的微型web开发框架。

最简Flask程序：

```
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return 'hello falsk'
```

然后在控制台中运行

```
export FLASK_APP=hello.py
#可选 export FLASK_DEBUG=1  进入debug模式。这样每次改动代码后就能直接浏览器显示更改效果，
flask run
```

Flask 依赖于 werkzeug 与 jinja 这两个核心库，werkzeug 是 HTTP 与 WSGI 相关的工具集，而 jinja 主要用于渲染前端模板文件。
Flask 框架满足 WSGI 协议，其功能简单而言就是将 HTTP 数据转为 environ 包含请求中所有的信息以及 start_response 回调函数传递给 web 框架对象。

这边查看下Flask的源码。

![image-20210109203329703](C:\Users\SheepHerder\AppData\Roaming\Typora\typora-user-images\image-20210109203329703.png)

注释大概意思就是flask对象继承了WSGI，作为整个web服务的中心，用于视图函数，url规则，模板配置等等。然后第一个参数是包的名字或者模块的名字。

![image-20210109204021079](C:\Users\SheepHerder\AppData\Roaming\Typora\typora-user-images\image-20210109204021079.png)

__init__方法没有什么好说的。仅从变量名就知道，它们用于存储每次请求对应的信息，相当于一个处理通道。
Flask 实例化后，接着利用 `@app.route('/')`装饰器的方式将 hello () 方法映射成了路由，相关代码如下：

![image-20210109204300118](C:\Users\SheepHerder\AppData\Roaming\Typora\typora-user-images\image-20210109204300118.png)

重点在endpoint，在flask中，endpoint是一个很重要的概念。

按照注释翻译，endpoint用来已经配置好的url,flask认为视图函数的名字作为endpoint。

个人喜欢将endpoint 翻译成 结点。

很多人认为：假设用户访问`http://www.AI.com/treemodel/xgboost`，`flask`会找到该函数，并传递`name='xgboost'`，执行这个函数并返回值。
但是实际中，`Flask`真的是直接根据路由查询视图函数么？
在源码中我们可以发现：

![image-20210109205121230](C:\Users\SheepHerder\AppData\Roaming\Typora\typora-user-images\image-20210109205121230.png)

![image-20210109205839932](C:\Users\SheepHerder\AppData\Roaming\Typora\typora-user-images\image-20210109205839932.png)

- 每个应用程序`app`都有一个**`view_functions`**，这是一个字典，存储`endpoint-view_func`键值对。

- route装饰器的作用和add_url_rule方法一致，只是用装饰器使用更高级点。

  ```
  @app.route('/')
          def index():
              pass
            等同于：
  def index():
      pass
  app.add_url_rule('/', 'index', index)
  
  ```

  ![image-20210109210938343](C:\Users\SheepHerder\AppData\Roaming\Typora\typora-user-images\image-20210109210938343.png)

- **`add_url_rule`的第一个作用就是向`view_functions`中添加键值对**(这件事在应用程序`run`之前就做好了)

- 每个应用程序`app`都有一个**`url_map`**，它是一个`Map`类(具体实现在`werkzeug/routing.py`中)，里面包含了一个列表，列表元素是`Role`的实例(`werkzeug/routing.py`中)。**`add_url_rule`的第二个作用就是向`url_map`中添加`Role`的实例**(它也是在应用程序`run`之前就做好了)

- 实际上，当请求传来一个url的时候，会先通过`rule`找到`endpoint`(`url_map`)，然后再根据`endpoint`再找到对应的`view_func`(view_functions)。通常，`endpoint`的名字都和视图函数名一样。
  这时候，这个`endpoint`也就好理解了：

  ```
  实际上这个endpoint就是一个Identifier，每个视图函数都有一个endpoint，
  当有请求来到的时候，用它来知道到底使用哪一个视图函数
  ```

  如果没有找到endpoint，也会自动将视图函数名称传入进去

  ![image-20210109211026432](C:\Users\SheepHerder\AppData\Roaming\Typora\typora-user-images\image-20210109211026432.png)

  在实际应用中，当我们需要在一个视图中跳转到另一个视图中的时候，我们经常会使用`url_for(endpoint)`去查询视图，而不是把地址硬编码到函数中。
  这个时候，我们就不能使用视图函数名当`endpoint`去查询了
  我们举个例子来说明。比如：

  ```python
  app = Flask(__name__)
  app.register_blueprint(user, url_prefix='user')
  app.register_blueprint(file, url_prefix='file')
  ```

  我们注册了2个蓝图。
  在user中(省略初始化过程)：

  ```python
  @user.route('/article')
  def article():
  	pass
  ```

  在file中(省略初始化过程)：

  ```python
  @file.route('/article')
  def article():
  	pass
  ```

  这时候，我们发现，`/article`这个路由对应了两个函数名一样的函数，分别在两个蓝图中。当我们使用`url_for(article)`调用的时候(注意，url_for是通过endpoint查询url地址，然后找视图函数)，`flask`无法知道到底使用哪个蓝图下的`endpoint`，所以我们需要这样:

  ```
  url_for('user.article')
  ```

一个 view function,可以有多个 endpoint、rule。是个一对多的关系。

 反过来,一个 endpoint,只能有一个 rule, 也只能有一个 view function。

这也是字典本身的特性！！！