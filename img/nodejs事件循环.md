# Nodejs的事件轮询
## 什么是事件轮询
事件轮询是nodejs处理非阻塞I/O事件的机制，虽然javascript是单线程，但是有时候会把一些操作转移到操作系统内核去处理。
操作系统内核都是多线程的，它们可以在后台处理多种操作，当其中一个事件处理完成的时候,内核会通知Nodejs，将该事件的回调函数添加到轮询队列里等待执行。
![内核通知nodejs](img/nodejs事件轮询—内核.jpg "nodejs事件轮询")

