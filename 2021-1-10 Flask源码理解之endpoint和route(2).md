2021-1-10 Flask源码理解之endpoint和route(2)



回归一下上一篇文章，在通过 @app.route () 装饰器将函数转为 Flask 视图函数时，多次提及了 endpoint，对应的 add_url_rule () 代码如下。

1. `# flask/app.py/Flask`
2. `def add_url_rule(self, rule, endpoint=None, view_func=None,``**options):`
3. `methods = options.pop('methods',``None)`
4. `rule = self.url_rule_class(rule, methods=methods,``**options)`
5. `self.url_map.add(rule)`
6. `if view_func is``not``None:`
7. `old_func = self.view_functions.get(endpoint)`
8. `if old_func is``not``None``and old_func != view_func:`
9. `raise``AssertionError('View function mapping is overwriting an '`
10. `'existing endpoint function: %s'``% endpoint)`
11. `self.view_functions[endpoint]``= view_func`



在 add_url_rule () 方法中通过 endpoint，将路由与视图函数关联在一起，为什么不直接将路由与视图函数关联？要多弄个 endpoint？

为了回答这个问题，先来了解一下 endpoint。

通常，可以通过两种方式将路由与视图函数关联。

1. `@app.route('/hello/<name>')`
2. `def hello(name):`
3. `return f'Hello, {name}!'`
4. `或`
5. `def hello(name):`
6. `return f'Hello, {name}!'`
7. `app.add_url_rule('/hello/<name>',``'hello', hello)`



本地运行起来，接着访问 `localhost:5000/hello/二两`，就会调用 hello () 方法，此时关联方式为：`localhost:5000/hello/二两` -> `endpoint:hello` -> `hello()方法`

这是最简单的写法，endpoint 与方法名相同，可以通过 endpoint 参数修改 endpoint 名称。

1. `@app.route('/hello/<name>', endpoint='sayhello')`
2. `def hello(name):`
3. `return f'Hello, {name}!'`



此时，关联改变为 `localhost:5000/hello/二两` -> `endpoint:sayhello` -> `hello()方法`。

通过 endpoint 可以快速构建 url，不再需要对 url 进行硬编码。

1. `@app.route('/')`
2. `def index():`
3. `# 将访问 hello/二两`
4. `print url_for('hello', name='二两')`



ok，ok，我明白了 endpoint 是做什么的，但还是一开始的问题，为什么要 endpoint？路由与函数直接对应上不就好了？

因为使用 endpoint 更方便，可以将所有后台管理的逻辑都放在 admin endpoint 下，将所用用户相关的放在 user endpoint 下，当然这要配合蓝图机制来使用。

1. `# main.py：`
2. `from flask import``Flask,``Blueprint`
3. `from admin import admin`
4. `from user import user`
5. `app =``Flask(__name__)`
6. `# 注册蓝图`
7. `app.register_blueprint(admin, url_prefix='admin')`
8. `app.register_blueprint(user, url_prefix='user')`
9. `# admin.py：`
10. `# 实例化蓝图`
11. `admin =``Blueprint('admin', __name__)`
12. `@admin.route('/home')`
13. `def home():`
14. `return``'Hello, root user!'`
15. `# user.py：`
16. `user =``Blueprint('user', __name__)`
17. `@user.route('/home')`
18. `def home():`
19. `return``'Hello, lowly normal user!'`
20. `# 使用时`
21. `print url_for('admin.home')``# Prints '/admin/home'`
22. `print url_for('user.home')``# Prints '/user/home'`



## Flask 路由机制

理解 endpoint 是理解 Flask 路由机制的前提，不然，当你浏览 Flask 路由机制匹配规则会比较蒙圈。

路由机制关键在于匹配，而匹配的逻辑在 dispatch_request () 方法中，该方法的调用路径为：`__call__()->wsgi\_app()->full_dispatch_request()->dispatch_request()`，方法代码如下。

1. `# flask/app.py`
2. `def dispatch_request(self):`
3. `req = _request_ctx_stack.top.request`
4. `if req.routing_exception is``not``None:`
5. `self.raise_routing_exception(req)`
6. `rule = req.url_rule`
7. `if getattr(rule,``'provide_automatic_options',``False) \`
8. `and req.method ==``'OPTIONS':`
9. `return self.make_default_options_response()`
10. `# 通过endpoint获得相应的视图函数`
11. `return self.view_functions[rule.endpoint](**req.view_args)`



从_request_ctx_stack 上下文中获得当前请求的上下文，找到当前请求的路由并从中找到 endpoint，再通过 endpoint 找到对应的视图函数。

关键在于 `req.url_rule`与 `rule.endpoint`这两个变量怎么来的？

_request_ctx_stack 变量涉及上下文相关的内容，细节先不提，其中存储着 RequestContext 类的实例对象，该对象与路由匹配相关的代码如下。

1. `# flask/ctx.py`
2. `class``RequestContext(object):`
3. `def __init__(self, app, environ, request=None, session=None):`
4. `self.app = app`
5. `if request is``None:`
6. `# 将environ转为request`
7. `request = app.request_class(environ)`
8. `self.request = request`
9. `self.url_adapter =``None`
10. `try:`
11. `self.url_adapter = app.create_url_adapter(self.request)`
12. `except``HTTPException``as e:`
13. `self.request.routing_exception = e`
14. `self.flashes =``None`
15. `self.session = session`
16. `# ... 省略无关代码`
17. `def match_request(self):`
18. `"""Can be overridden by a subclass to hook into the matching`
19. `of the request.`
20. `"""`
21. `try:`
22. `# 匹配url`
23. `result = self.url_adapter.match(return_rule=True)`
24. `# 返回结果`
25. `self.request.url_rule, self.request.view_args = result`
26. `except``HTTPException``as e:`
27. `self.request.routing_exception = e`



在__init__() 中，通过 app.request_class () 方法，将 environ 转为 Request 类实例，接着使用 app.create_url_adapter () 方法将 request 相关信息存到 url_map 变量中。

在路由匹配时会调用 match_request () 方法，该方法具体的匹配规则又由 self.url_adapter.match () 方法完成，该方法会返回 url_rule 与 view_args。

为了进一步理解，剖析一下 create_url_adapter () 方法与 match () 方法，先看 create_url_adapter () 方法，一层层看下去，该方法的调用顺序为：`create_url_adapter()->self.url_map.bind_to_environ()->Map.bind()->MapAdapter()`，简单而言，该方法最后返回一个 MapAdapter 类实例，MapAdapter 类下就有 match () 方法，上面 self.url_adapter.match () 调用的就是这个方法，该方法实现具体的匹配逻辑，最终返回返回 url_rule 与 view_args (路由与视图函数的参数)

1. ```
   1. `# werkzeug/routing.py/MapAdapter`
   2. `def match(self, path_info=None, method=None, return_rule=False, query_args=None):`
   3. `	for rule in self.map._rules:`
   4. `		try:`
   5. `			rv = rule.match(path, method)`
   6. `		except:`
   7. `# ... 省略`
   8. `# 返回 路由与视图函数的参数`
   9. `			if return_rule:`
   10. `				return rule, rv`
   11. `			else:`
   12. `				return rule.endpoint, rv`
   
   
   ```

   

match () 匹配规则的逻辑比较细节，有兴趣的可以去 werkzeug 的 routing.py 文件中查阅。

至此，Flask 路由的大致过程就分析完了，简单总结一下：

- \1. 通过 @app.route 装饰器或者 app.add_url_rule () 方法注册视图函数
- \2. 每次请求时，都会利用上下文的形式将路由匹配结果存储起来。匹配逻辑最终由 MapAdapter 类的 match () 方法完成。
- \3. 最后，通过 dispatch_request () 方法获取此前匹配的路由结果，调用相应的视图函数

over！