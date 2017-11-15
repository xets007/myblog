---
title: '《Flask Web Development》Book Review'
date: 2016-07-14 10:23:30
categories: Python
tags:
- Book
- Flask
---

这本书是偶然在一篇博客[《如果有人让你推荐Python技术书，请让他看这个列表》](http://python.jobbole.com/85620/)中看到的，《Flask Web Development》被列在进阶级，豆瓣评分8.6，算是比较高，比较受公众认可的了。正好最近工作中要用到Web开发相关知识，花了两天时间看了下这本书，果然没让人失望。

这是一本写作技巧和Web技术并重的书，从第一页开始就让你爱不释手，想一口气读完，酣畅淋漓。

<!-- more -->

作者循循善诱，先抛出一个问题，然后给出一种解决方案，有时候这种解决方案是最佳的，有时候并不完美，有副作用，这也是作者刻意安排的，因为作者又要抛出另外一个问题，引出另外一个有意思的话题。时不时一语道破天机，让你自然而然地接受知识。读书的过程中你就会感受到，隐隐有一根主线，把所有零碎的知识都串联起来。把复杂抽象的知识通过简单具体的方式呈现，娓娓道来。

作者对书中内容字斟句酌，精心布局，如果没有长年累月的积累，没有深入的实践与思考，是达不到这样的高度的。

也许《Flask Web Development》这本书中讲的Flask Web开发，你以后不一定会用到，但书中内容的编排，行文的方式，一定对你写文章、写书大有帮助。

最后，感受下Flask的简洁优美，领悟Flask是怎样建立生态的。下面列出了《Flask Web Development》中使用到的Flask扩展：

```sh
$ pip list | grep Flask
Flask (0.10.1)
Flask-Bootstrap (3.3.6.0)
Flask-HTTPAuth (3.1.2)
Flask-Login (0.3.2)
Flask-Mail (0.9.1)
Flask-Migrate (1.8.1)
Flask-Moment (0.5.1)
Flask-PageDown (0.2.1)
Flask-Script (2.0.5)
Flask-SQLAlchemy (2.1)
Flask-WTF (0.12)
```

再看看各个模块是怎么组织起来的：
```python
from flask import Flask
from flask.ext.bootstrap import Bootstrap
from flask.ext.mail import Mail
from flask.ext.moment import Moment
from flask.ext.sqlalchemy import SQLAlchemy
from flask.ext.login import LoginManager
from flask.ext.pagedown import PageDown
from config import config

bootstrap = Bootstrap()
mail = Mail()
moment = Moment()
db = SQLAlchemy()
pagedown = PageDown()

login_manager = LoginManager()
login_manager.session_protection = 'strong'
login_manager.login_view = 'auth.login'


def create_app(config_name):
    app = Flask(__name__)
    app.config.from_object(config[config_name])
    config[config_name].init_app(app)

    bootstrap.init_app(app)
    mail.init_app(app)
    moment.init_app(app)
    db.init_app(app)
    login_manager.init_app(app)
    pagedown.init_app(app)

    if not app.debug and not app.testing and not app.config['SSL_DISABLE']:
        from flask.ext.sslify import SSLify
        sslify = SSLify(app)

    from .main import main as main_blueprint
    app.register_blueprint(main_blueprint)

    from .auth import auth as auth_blueprint
    app.register_blueprint(auth_blueprint, url_prefix='/auth')

    from .api_1_0 import api as api_1_0_blueprint
    app.register_blueprint(api_1_0_blueprint, url_prefix='/api/v1.0')

    return app  
```
