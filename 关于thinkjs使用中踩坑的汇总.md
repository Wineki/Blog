### 关于Thinkjs使用中踩坑的汇总--链式调用问题
=============

第一篇博文，由于刚开始学习markdown的方式写博文，速度好慢。<br/>
最近在用*Thinkjs*写项目，又向从前端到全段的梦想迈进了一步，哈哈，写话少说，share一些干货粗来。<br/>
*Thinkjs*是基于nodejs的web框架，具体的文档可以参看<https://thinkjs.org/>官方文档，我使用的是1.0版本，所以暂时只讨论1.0的一些情况。
今天写项目中遇到这样一个问题：
我从session中获取到用户的信息，然后获取存在session中的userName，使用：

	this.session("userInfo").then(function(data){
	...
	});

的方式，另一方面我通过数据库查询获取到用户在数据库中存储的userName,然后比较这两个userName时候否相同，使用

	D('User').get(id).then(function(data){
	...
	}); 


再将一个标签值assign到view层对应的页面。
然而就在编写的过程中我发现我在比较之后，无法将标签值传入页面。
具体代码如图：
![1](http://p6.qhimg.com/d/inn/f451a883/code.png)
这是一份伪代码，上面的图片的大概意思就是这样的：

	return D('User').get(id).then(function(data){
		return self.session('user').then(function(user){
			//验证session信息,返回editable
			self.assign('editable',editable);
			self.assign('user',data);
			self.display();
		});
	});
最初我是这么写的：

	return D('User').get(id).then(function(data){
		var session = self.session('user').then(function(user){
			//验证session信息,返回editable
			self.assign('editable',editable);
		});
		self.assign('user',data);
		self.display();
	});
然后我发现这个根本没有想下执行，页面直接白屏了，原因其实是这样的，在*thinkjs*中绝大多数方法都是异步调用的，并且包装成promise，而promise是通过
*try{}catch{}*进行异常捕获的，很显然我没有catch异常所以导致直接白屏，同时由于thinkjs中许多的调用都是异步调用（这个之前说过），也就是说在session还没有返回值的时候程序就向下执行了，但是此处我必须要它同步来执行，也就是说按照程序写的先后顺序来执行，所以就采用return的方法，将没有个异步方法串联起来，形成一个调用链就搞定了。


