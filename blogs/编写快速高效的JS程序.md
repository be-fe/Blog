## 编写快速高效的JS程序
[原文链接](http://www.smashingmagazine.com/2012/11/05/writing-fast-memory-efficient-javascript/)   
对于像Google的V8这样专门为大型JS程序设计的引擎来说，如果你在开发中注重内存的使用和控制，那么就必须了解浏览器的JS引擎背后的一些东西。

### JS在V8上如何运行？
如果你对JS引擎没有较深的了解，开发一个大型Web应用也没啥问题，就像司机们只是告诉你他们曾不止一次的看过汽车的引擎盖。Chrome浏览器是我的选择，这里我将谈下V8的核心组件：

- 基础编译， 解析你的JS代码，并且在其执行之前生成原生的机器码，在真正执行之前，这个代码是没有经过优化的。
- V8将你定义的对象声明为对象模型，对象在JavaScript语言中被声明为关联数组，但是在V8中，他们被声明为hidden classes，这是一个用于优化的内部类型系统。
- 执行解析器会监听这个系统，并且识别出那些‘hot’函数。
- 优化编译器会重新编译并优化“hot”代码
- V8支持免优化，也就是说当你发现一些优化代码并非所需，那么可以选择不进行优化。
- JS有一个垃圾收集器，理解垃圾收集机制对于优化JavaScript代码非常重要。

### 垃圾收集

![](http://www.flickr.com/photos/26817893@N05/2864644153/)


### 错误的清除引用
网上很多讨论JavaScript内存回收的讨论中，一个关键词`delete`常被提起，它可以用于删除一个map中的key，一些开发者认为可以这样就可以强行的删除引用，而事实上应当避免使用`delete`,比如下面这个例子：

    var o = { x: 1 }; 
    delete o.x; // true 
    o.x; // undefined
    
`delete o.x`在这个场景下实际上做了更糟糕的事情，它改变了`o`的hidden class，并使之成为了一个没有hidden class的更慢的对象。

即便如此，你可以发现许多流行的JS类库中都使用了`delete`




同样的错误观念存在于null的使用上，将一个对象设置为null并不会"null"这个对象，它会将这个对象的引用指向null，使用`o.x = null`比使用`delete`要好，但是多数情况下这并没有必要。

    var o = { x: 1 }; 
    o = null;
    o; // null
    o.x // TypeError
如果这是该对象的最后一次引用，这个对象就会被回收，如果不是，这个对象就仍然是被访问的状态，就不会被回收。

另一个要点是全局变量在页面的生命周期内是无法被清除的。只有这个页面处于打开状态，这个变量就会一直存在。

    var myGlobalNamespace = {};
全局变量会在刷新页面，指向另一个不同的页面，关闭标签页或者直接关闭浏览器时才会被清空，对于函数作用域内的变量，当函数执行完毕并且没有任何东西引用它时，变量就会被清除。

### 经验法则
为了让垃圾收集器尽可能早以及尽可能多的清除对象，只要不持有不需要的对象，大多数情况下收集器会自动帮你解决问题，不过还是有一些事情需要考虑。

- 最好的手动清除方式是在合适的作用域下使用变量。也就是说，尽量使用函数内的变量，只要该函数不再被引用，变量就会被清除。
- 清除不必要的事件监听器，尤其是在需要移除DOM对象的时候。
- 如果你使用本地的数据缓存，确认及时的清除它，避免用不到的大段数据被存储。

### 函数
下面，让我们来看看函数，就像我们刚才讲的那样，当一段内存不再会被使用时，垃圾收集就会被处罚，为了更好的说明，请看下面的一些例子。

    function foo() {
        var bar = new LargeObject();
        bar.someCall();
    }
 当函数foo return， bar这个对象就会自动被回收，因为没有任何东西区引用它了。
 再拿下面这个做一个比较：
 
     function foo() {
        var bar = new LargeObject();
        bar.someCall();
        return bar;
    }

    // somewhere else
    var b = foo();
    
现在一个外部变量`b`引用了`bar`这个对象，那么`bar`就不会被回收。 

### 闭包
如果一个函数return了一个在它内部定义的函数，当这个函数执行后，这个内部函数就可以访问外部的作用域。这样就形成了一个闭包 —— 一个拥有特殊上下文环境的表达式。

    function sum (x) {
        function sumIt(y) {
            return x + y;
        };
        return sumIt;
    }

    // Usage
    var sumA = sum(4);
    var sumB = sumA(3);
    console.log(sumB); // Returns 7

`sum`函数内的变量都不会被垃圾回收，因为它们都被一个全局变量所引用，它仍然可以通过`sumA(n)`进行访问。
我们再看另一个例子，我们能访问到`largeStr`么?

    var a = function () {
        var largeStr = new Array(1000000).join('x');
        return function () {
            return largeStr;
        };
    }();

是的，我们可以，它没有被垃圾收集，那么下面这个呢？

    var a = function () {
        var smallStr = 'x';
        var largeStr = new Array(1000000).join('x');
        return function (n) {
            return smallStr;
        };
    }();

We can’t access it anymore and it’s a candidate for garbage collection.

### 定时器
循环或者setTimeout()/setInterval()是最容易出现内存泄露的地方，并且普遍存在。

想一下下面这个例子：

    var myObj = {
        callMeMaybe: function () {
            var myRef = this;
            var val = setTimeout(function () { 
                console.log('Time is running out!'); 
                myRef.callMeMaybe();
            }, 1000);
        }
    };

如果我们执行：
    
    myObj.callMeMaybe();
可以发现每秒钟控制台都会输出 'Time is running out!'，接着执行：

    myObj = null;

定时器此时仍然在，`myObj`并不会被垃圾收集,因为`setTimeout`中的闭包仍然需要执行。myObj中的myRef仍然被引用.
这对于setTimeout/setInterval内部的引用同样有指导意义，例如一些函数，需要在它们被垃圾回收之前执行完成。

### 需要注意的性能陷阱
很重要的一点是，除非你真正需要，否则没有必要优化你的代码，这个怎么强调都不为过。在大量的微基准测试中，你可以很轻易地发现，在V8引擎中N比M更加的优化，但是如果在真实的代码模型或者在真正的应用程序中进行测试，那些优化的实际影响可能比你期望的要小得多。  
假设现在我们想要建立的一个模块：

- 通过数字ID取出本地存储的数据资源。

- 用获得的数据生成表格内容。

- 为每个表格单元添加事件处理，每当用户点击表格单元，切换表格单元的class。

即使这个问题解决起来很直观，但是有一些困难的因素。我们如何去存储这些数据，如何可以高效地生成一个表格并把它添加到DOM中去，如何优化地处理这个表格的事件处理？

第一个（也是幼稚的）采取的方案可能是将每块可获取的数据存放在一个对象中，然后把所有对象集合到一个数组当中。有的人可能会用jQuery去循环访问数据然后把生成表格内容，然后把它添加到DOM中。最后，可能会用使用事件绑定添加点击我们需要的点击事件。

注意：这不是你应该做的事情：

    var moduleA = function () {

        return {
            data: dataArrayObject,
            init: function () {
                this.addTable();
                this.addEvents();
            },
            addTable: function () {
                for (var i = 0; i < rows; i++) {
                    $tr = $('<tr></tr>');
                    for (var j = 0; j < this.data.length; j++) {
                        $tr.append('<td>' + this.data[j]['id'] + '</td>');
                    }
                    $tr.appendTo($tbody);
                }
            },
            addEvents: function () {
                $('table td').on('click', function () {
                    $(this).toggleClass('active');
                });
            }
        };
    }();
代码简单，但它完成了我们需要的工作。

在这种情况下，我们唯一要迭代的只是ID，在一个标准的数组当中，数字属性可以更简单地表示出来。有趣的是，直接用DocumentFragment和原生的DOM方法生成表格内容，比你用jQuery（上面的jQuery用法）更加的优化。当然，使用事件委托通常比为每个td都进行事件绑定会有更好的性能。

注意jQuery内部确实使用DocumentFragment进行了优化，但在我们的例子中，代码中在循环中调用append（），每一次调用都要进行额外的操作，所以在这个例子中，它达到优化效果可能并不大。希望这应该不会是一个痛处，但是一定要用基准测试来确保自己的代码没有问题。

在我们的例子当中，添加这些以上的优化会得到一些不错（预期）的性能收益。相对于简单的绑定，事件委托提供了相当好的改进，且选择用documentFragment会是一个真正的性能助推器。

    var moduleD = function () {

        return {

            data: dataArray,

            init: function () {
                this.addTable();
                this.addEvents();
            },
            addTable: function () {
                var td, tr;
                var frag = document.createDocumentFragment();
                var frag2 = document.createDocumentFragment();

                for (var i = 0; i < rows; i++) {
                    tr = document.createElement('tr');
                    for (var j = 0; j < this.data.length; j++) {
                        td = document.createElement('td');
                        td.appendChild(document.createTextNode(this.data[j]));

                        frag2.appendChild(td);
                    }
                    tr.appendChild(frag2);
                    frag.appendChild(tr);
                }
                tbody.appendChild(frag);
            },
            addEvents: function () {
                $('table').on('click', 'td', function () {
                    $(this).toggleClass('active');
                });
            }

        };

    }();
我们可能会寻找其他的方案来提高性能。你可能在某些文章中了解到用原型模式比用模块模式更加优化（我们不久前已经证明了事实并非如此），或者了解到JavaScript模板框架是经过高度的优化的。有时它们的确是这样，但是使用它们只是为了代码拥有更强的可读性。同时，还有预编译！让我们测试一下，实际上这有多少是能带来真正优化的。

    moduleG = function () {};

    moduleG.prototype.data = dataArray;
    moduleG.prototype.init = function () {
        this.addTable();
        this.addEvents();
    };
    moduleG.prototype.addTable = function () {
        var template = _.template($('#template').text());
        var html = template({'data' : this.data});
        $tbody.append(html);
    };
    moduleG.prototype.addEvents = function () {
       $('table').on('click', 'td', function () {
           $(this).toggleClass('active');
       });
    };

    var modG = new moduleG();
正如结果所示，在这种情况下所带来的性能效益是微不足道的。选择模板和原型不会真正提供得比我们原来拥有的东西更多的东西。据说，性能并不是现代开发者所真正使用它们的原因——而是它给你的代码库所带来的可读性，继承模型，以及可维护性。  

### V8优化技巧

同时详细的陈列每一个V8的每一种优化显然超出了本文的讨论范围，其中有许多特定的优化技巧都值得注意。记住以下的一些建议你就可以减少你写出低性能的代码的机会。

- 特定的模式会导致V8放弃优化。例如使用try-catch，就会导致这种情况的发生。如果想要了解跟多关于什么函数可以被优化，什么函数不可以，你可以使用V8引擎中附带的D8shell实用程序中的–trace-optfile.js。

- 如果你关心运行速度，那么就要尽量保持你的函数的功能的单一性，也就是说，确保变量（包括属性，数组，和函数参数）永远只是相同隐藏类的包含对象。例如，永远不要干这种事：
 
    function add(x, y) { 
       return x+y;
    } 

    add(1, 2); 
    add('a','b'); 
    add(my_custom_object, undefined);

- 不要加载未声明的或者已经被删除的元素。这样做可能对你的程序运行结果不会造成影响。但是它会使得程序运行得更慢。

- 不要写过于庞大的函数，因为他们更难被优化。

如果想知道更多的优化技巧，可以观看DanielClifford的GoogleI/O大会上的演讲BreakingtheJavaScriptSpeedLimitwithV8，它同时也涵盖了上面我们所说的优化技巧。OptimizingForV8—ASeries也同样值得一读。