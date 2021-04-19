

### WSGI
* 全称是Web Server Gateway Interface，WSGI不是服务器，python模块，框架，API或者任何软件，只是一种规范，描述web server如何与web application通信的规范。要实现WSGI协议，必须同时实现web server和web application，当前运行在WSGI协议之上的web框架有Bottle, Flask, Django等

### uwsgi
* 与WSGI一样是一种通信协议，是uWSGI服务器的独占协议，用于定义传输信息的类型(type of information)，每一个uwsgi packet前4byte为传输信息类型的描述，与WSGI协议是两种东西

### uWSGI
* 是一个web服务器，实现了WSGI协议、uwsgi协议、http协议等




### 附录
* 参考1：https://www.jianshu.com/p/679dee0a4193
* 参考2：https://zhuanlan.zhihu.com/p/66144617
* 参考3：https://www.liaoxuefeng.com/wiki/1016959663602400/1017805733037760
* 参考4：https://www.jianshu.com/p/679dee0a4193

https://docs.python.org/3/library/wsgiref.html#
