## 软件工程（3） - 微信抢票实战

**目录**

- 环境准备
  - 服务器部署
  - 程序环境
  - Git项目
- 初识原理
  - Handler和Wrapper
  - Views
  - Models
- 测试方法
  - 单元测试
- 其他值得说的地方
  - 装饰器
  - 去除外键

### 环境准备

#### 服务器部署

首先感谢腾讯爸爸的服务器~

系统是ubuntu 16纯通过控制台操作，实现下来确实了解了好多终端操作的知识！个人认为MobaXterm这一款终端比较好用，安利一下

![mobaXterm](.\image\mobax.png)

部署流程是参考这一篇著名博客完成的，推荐大家阅读How to setup my first server

#### 程序环境

使用venv虚拟环境，python版本是3.5（必须是3.5，因为3.6版本对于框架中的部分代码不支持），框架是Django项目。

在本地我们使用PyCharm进行代码的编辑（因为专门支持python以及Django项目的IDE对于代码补全、调试和框架等都有很好的支持），此外PyCharm提供直接与服务器通信传输文件的方式，使同步代码简单了很多。在Tools -> Deployment -> Configuration内进行服务器连接的配置即可。

#### Git项目

该项目使用Gitlab进行项目管理，项目地址为gitlab/WeChatTicket。主要在`dev_sgy`分支上进行开发和一些同步，总体上还是有一棵比较漂亮的分支树！

![branch](.\image\branch.png)

---

### 初识原理

#### Handler和Wrapper

项目的结构是这样的：

![structure](.\image\struc.png)

其中wechat、userpage、adminpage是项目下的3个app，在wechat中是微信显示出的界面及其逻辑层。在类`CustomWeChatView`中，handler的列表如下所示：

```python
handlers = [
        BookHandler,
        HelpOrSubscribeHandler,
        FormulaHandler,
        UnbindOrUnsubscribeHandler,
        QuitBookHandler,
        BindAccountHandler,
        ActivityListHandler,
        TicketListHandler,
        BookEmptyHandler,
    ]
```

而每个Handler都需要实现两个方法

```python
class BookHandler(WeChatHandler):
  def check(self):
    # ...
  def handle(self):
    # ...
```

即，按顺序查找Handler，如果它的`check()`方法返回真，就相当于进入其分支，并执行`handle()`方法来进行操作，也不会进入其他分支。

#### Views

像一般Django项目中的那样，我们需要将view与一个url地址绑定，例如我们定义

```python
class AdminLogin(APIView):
  def get(self):
    # ...
  def post(self):
    # ...
```

就需要在urls.py中绑定它，即

```python
url(r'^login/?$', AdminLogin.as_view())
```
程序已经提供了`self.input[]`作为传输数据的接口，利用这个接口就可以执行逻辑操作。

#### Models

这是后端项目中Django设置的与数据库交互的方法。在上一个Django项目——活动管理网站中，我们使用的大多是原生的SQL语句，即调用`Model.object.raw()`方法。这样做对于不熟悉数据库直接操作的同学来说有一些难以使用，也没有发挥出Django的优势。Django自带的QuerySet是很好用的选择方法，支持很多种类的查找与统计。例如：

```python
def get_used_activity_count_by_id(cls, activity_id):
    return cls.objects.filter(
        activity_id=activity_id,
        status__exact=cls.STATUS_USED
    ).count()
```

事实上在执行时会自动地转化成取满足条件的行数（由于我们的本意即只是想求出数量，而不需要查出所有的数据，这样无疑是大幅减少查找时间的。）

更多详细关于模型查找的讲解可以在自强学堂中找到。[Django-QuerySet](http://code.ziqiangxuetang.com/django/django-queryset-api.html)

### 测试方法

#### 单元测试

单元测试是新学习的知识之一。我们知道运行这个Django项目是需要服务器支持的，在服务器上我们部署了服务器，但是本地的环境并不容易配置，使用PyCharm自带的localhost运行效果差强人意，其他的就更难以测试。所幸它提供了Django Test模板，可以在app下的`tests.py`中创建该app的单元测试，以测试某一个视图逻辑的正确性。

我们将需要：

```python
from django.test import TestCase
```

这是Django自带的测试库。最大的优点即是它可以仿造数据库，在没有真实数据库的情况下假定出Model对象。事实上，就算是本地配置好数据库环境，为了测试而添加的数据也可能对其造成毁坏，因此这种仿制的方法既不损害本地数据库，又能便捷地形成我们需要的对象（无需SQL语句操作）。重点便是使用`Mock()`函数。

有趣的是，`Mock()`函数返回一个对象，这个对象只需要**被测试的属性与真实Model相同**即可。在本例中，一个活动有许多值，但是如果我们只需要测试其`user`部分，便只在`Mock()`对象中声明其`user`值即可。这体现出了测试的优越性。

```python
request.body.decode = Mock(return_value='{"id": "1"}')
with patch.object(Activity, 'get_by_id', return_value=f_Activity()):
    response = json.loads(found.func(request).content.decode())
    self.assertEqual(response['code'], ERROR_TYPE_NORMAL)
```

在上例中我们只需要测试它的id字段，那么可以如此高效地形成用于测试的Model对象。

### 其他值得说的地方

#### 装饰器

python有自带的一些装饰器，如Django支持便捷的用户登录状态检测

```python
@login_required
```

但是对于这个项目中的view类，它的user并不直接存在与成员中，而是传输的一个参数，这样原生的装饰器就不能使用。经过研究我编写了自己的装饰器：


```python
def m_login_required(view):
    def wrapper(*args, **kargs):
        if not args[0].request.user.is_authenticated():
            raise LogicError('User not online.')
        return view(args[0], **kargs)

    return wrapper
```

由于user包含于`view.request.user`中，而又利用了成员方法的第一个参数为`self` ，使用这种方式来实现装饰器。装饰器的原理就是为目标函数包裹一个外壳，在其之外再绑定其他的一些代码段。

#### 去除外键

这是文档中提出的一个优化方案。确实，将活动作为电子票的外键的意义不大，只是减少了一次调用，但是时间成本是并没有减少的。去除外键后显而易见地，用活动的id作为索引即可查找到活动本身，更何况有很多情况我们只需要传送活动的id。修改后的电子票模型如下：

```python
class Ticket(models.Model):
    student_id = models.CharField(max_length=32, db_index=True)
    unique_id = models.CharField(max_length=64, db_index=True, unique=True)
    activity_id = models.IntegerField()
    status = models.IntegerField()

    STATUS_CANCELLED = 0
    STATUS_VALID = 1
    STATUS_USED = 2
```

虽然没有实际测试，但是对开发者来说这样的写法会更加友好。