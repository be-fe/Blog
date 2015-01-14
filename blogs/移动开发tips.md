##移动开发Tips
### 滚动问题
iOS 5和安卓4.1开始支持div内滚动条：

    -webkit-overflow-scrolling: touch; 
    overflow: auto;    
老版本尽量使用body滚动条或者采用iscroll
### backgroundp-size兼容性
安卓4.4版本之前不支持将backgroundp-size与background合并，如果在background中简写background-size，整条语句就失效了，gulp的css-min压缩会将两个属性合并，因此需要添加如下配置项：

    minifyCSS({'noAdvanced': true})
    
### iOS7的header问题
header会覆盖在webview上层，其他版本则没有这个问题。

### 动画性能
主要有3个相关维度影响了动画性能：
- **分辨率：**高分辨率的机器更吃性能。我的小米2s滑动还不如我的安卓2.2的老机器。
- **webview版本：**安卓4.4以前的webview是一个简化版的webkit内核，性能很差，4.4以后采用了chromium的内核性能有了很大的提升。
- **硬件性能：**硬件性能越好，当然对流畅度越好。

### hover效果
移动端是没有hover效果的，如果在CSS中有如下代码：

    .test{color:red;}
    .test:hover{color:blue;}
在某些浏览器下，比如iOS的Safari，实际生效的会是第二行代码。也就是说，该元素的颜色实际上会一直保持蓝色。
### flex布局
移动端可以使用flex进行布局，但是需要注意flex的语法标准是修订过一次的，一部分浏览器使用的是老的语法，一部分采用新的语法，因此需要注意兼容性问题，比如需要像下面这样的写法：

    .row {
      display: -webkit-box;
      display: -webkit-flex;
      display: -moz-box;
      display: -moz-flex;
      display: -ms-flexbox;
      display: flex;
      padding: 5px;
      width: 100%; }
### 300ms延迟
移动端的click事件有300ms的延迟，一方面可以通过touch事件的组合来模拟一个tap事件来代替click（很多库中都有对tap的封装），另外在Safari和Android chrome 32以后的版本上已经移除了click事件的延迟，随着浏览器的发展，或许几年后可以逐步放弃模拟的方式。

### safari的alert存在bug
如果存在这样的代码结构：

    $("ele").click(function(){
        if(condition){
            do ...
            alert("success")
        }
        else{
            do ...
            alert("failed")
        }
    })
最终if和else中的逻辑都会执行，而去掉alert后则恢复正常。只有在手机safari中发现该问题，在其它浏览器中没发现这样的问题。
### 覆盖 WebView 默认的样式
    body{
	-webkit-touch-callout: none;	/*在iOS浏览器里面，假如用户长按a标签，都会出现默认的弹出菜单事件*/
	-webkit-text-size-adjust:none;	/* 字型大小是不會變的，而使用者就無法透過縮放網頁來達成放大字型 */
	-webkit-appearance: none;		/*可以改变按钮或者其它控件看起来类似本地的控件*/
	-webkit-tap-highlight-color: transparent;/*Mobile上点击链接高亮的时候设置颜色为透明*/ 
	-webkit-user-drag: none; 		/*-webkit-user-drag CSS 属性控制能否将元素作为一个整体拖动。*/
    }
    a{
    	-webkit-tap-highlight-color: rgba(0, 0, 0, 0); /*很多Android 浏览器的 a 链接有边框，这里取消它*/
    }
    /* For WebApp */
    body{  
      -webkit-user-select : none; /* 如果用户长按web网页的文本内容，都会出现选中的行为，提供复制文字等功能。禁止用户选中文字 for iOS */
    } 
    a, img{  
      -webkit-touch-callout:none;  /* 禁止用户在新窗口打开页面、如何禁止用户保存图片＼复制图片 for iOS */  
    } 
    
### iOS 8 box-shadow不显示
解决方法：

    border-radius: 1px;
### Android4.4 滑动操作时会触发touchcancle
Android4.4以上版本改换了浏览内核，在touchstart之后，如果发现你进行滑动操作，那么就会触发touchcancle，后续绑定在touchmove和touchend上的回调就不会再执行了。

### Android2.3 innerHTML不能用于赋值页面元素
Android2.3系统下，对页面上存在的元素做innerHTML赋值会报错，但是可以createElement出来的元素使用innerHTML赋值。jQuery的html()方法可以兼容。

### Android2.3 border radius不支持百分比
如果要构造圆形，可以使用

    border-radius: 9999px;