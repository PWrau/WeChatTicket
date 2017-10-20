软件工程（3） - 微信抢票实战

目录

- 环境准备
  - 服务器部署
  - 程序环境
  - Git项目
- 初识原理
  - Handler和Wrapper
  - Views



环境准备

服务器部署

首先感谢腾讯爸爸的服务器~

系统是ubuntu 16纯通过控制台操作，实现下来确实了解了好多终端操作的知识！个人认为MobaXterm这一款终端比较好用，安利一下

![mobaXterm](E:\Work\2017Autumn\SoftwareEngineering\LiuQiang\抢票实战\mobax.png)

部署流程是参考这一篇著名博客完成的，推荐大家阅读How to setup my first server

程序环境

使用venv虚拟环境，python版本是3.5（必须是3.5，因为3.6版本对于框架中的部分代码不支持），框架是Django项目。

在本地我们使用PyCharm进行代码的编辑（因为专门支持python以及Django项目的IDE对于代码补全、调试和框架等都有很好的支持），此外PyCharm提供直接与服务器通信传输文件的方式，使同步代码简单了很多。在Tools -> Deployment -> Configuration内进行服务器连接的配置即可。

Git项目

该项目使用Gitlab进行项目管理，项目地址为gitlab/WeChatTicket。主要在dev_sgy分支上进行开发和一些同步，总体上还是有一棵比较漂亮的分支树！

![gitlabbranch](E:\Work\2017Autumn\SoftwareEngineering\LiuQiang\抢票实战\branch.png)

---

初识原理

Handler和Wrapper

项目的结构是这样的：

![structure](E:\Work\2017Autumn\SoftwareEngineering\LiuQiang\抢票实战\struc.png)

其中wechat、userpage、adminpage是项目下的3个app，在wechat中是微信显示出的界面及其逻辑层。在类CustomWeChatView中，handler的列表如下所示：

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

而每个Handler都需要实现两个方法

    class BookHandler(WeChatHandler):
      def check(self):
        # ...
      def handle(self):
        # ...

即，按顺序查找Handler，如果它的chheck()方法返回真，就相当于进入其分支，并执行handle()方法来进行操作，也不会进入其他分支。

Views

像一般Django项目中的那样，我们需要将view与一个url地址绑定，例如我们定义

    class AdminLogin(APIView):
      def get(self):
        # ...
      def post(self):
        # ...

就需要在urls.py中绑定它，即

    url(r'^login/?$', AdminLogin.as_view())

程序已经提供了self.input[]作为传输数据的接口，利用这个接口就可以执行逻辑操作。
