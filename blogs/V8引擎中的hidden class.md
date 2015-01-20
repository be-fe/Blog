## V8引擎中的hidden class
[原文链接](https://developers.google.com/v8/design?csw=1#prop_access)

hidden class是为了实现对象属性的快速存取。

JavaScript是一种动态编程语言：属性可进行动态的添加和删除，这意味着一个对象的属性是可变的，大多数的JavaScript引擎使用一个类似字典的数据结构来存储对象的属性 —— 那么每个属性的访问都需要动态的去查询属性在内存中的位置。那么相比Java这样的静态语言来说就会慢的多。静态语言的属性地址会在类定义后通过编译，相对于对象有一个固定的偏移量，访问属性本质上只是简单的读取和存储，一行命令就可以搞定。

为了提高属性的访问速度， 在这种场景下，V8并没有动态的去内存中查询属性，而是动态的去创建hidden class。hidden class的基本原理并非什么新的创见，基于原型的编程语言都会采用类似的方式来处理maps。在V8引擎中，当一个新的属性加入时，对象就会改变自己的hidden class。

例如这样一个简单的函数：

    function Point(x, y) {
      this.x = x;
      this.y = y;
    }
当你执行`new Point(x, y)`后， 一个新的Point的实例对象被创建了，首先V8会创建一个初始的Point的hidden class，在这个例子中，我们叫它`C0`. 这时这个对象中还没有任何属性被定义，初始的hidden class为空。这一阶段，Point对象的hidden class是`C0`.
![](http://text-learn.qiniudn.com/map_trans_a.png)

然后执行第一条语句：`Point (this.x = x;)`,这时会创建一个新的属性`x`，在这个案例中，V8会执行：

- 基于`C0`创建另一个hidden class `C1`，并在`C1`中描述这个对象有一个属性`x`，它的值存储位置在Point对象的offset 0 上

- 当属性 `x` 已经被加入这个对象后，`C1` 会代替 `C0` 成为这个对象的hidden class。

![](http://text-learn.qiniudn.com/map_trans_b.png)
接着程序会执行第二条语句`Point (this.y = y;)`, 创建一个新的属性 `y`, 类似的，V8会执行： 

- 基于`C1`创建另一个hidden class `C2`, 并在`C2`中描述这个对象还有一个属性`y`，它的值存储位置在Point对象的offset 1 上

- 当属性 `y` 已经被加入这个对象后，`C2` 会代替 `C1` 成为这个对象的hidden class。

![](http://text-learn.qiniudn.com/map_trans_c.png)
在添加属性时同时创建一个新的hidden class看起来很低效。不过因为hidden class是可以重用的，下一次再new一个Point的实例的时候，就不会重复的创建hidden class了。

如果删除对对象中的某个属性执行了`delete`操作，那么就无法通过hidden class进行访问寻址了  

总结一下，hidden class实际是一种空间换时间的思路，通过为对象设置hiddden class，标记出了对象属性值存储位置相对于object的指针偏移，从而实现快速的访问属性值。  

#### 相关阅读
[貘大关于hidden class的文章](http://www.w3ctech.com/topic/660)   
[测试delete属性改变hidden class](http://jsperf.com/test-v8-delete)   
[示例](http://debuggable.com/posts/understanding-hidden-classes-in-v8:4c7e81e4-1330-4398-8bd2-761bcbdd56cb)