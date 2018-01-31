#手机摇一摇、刮刮乐

 

[TOC]

## 整体思路

1. 监听devicemotion事件，获取accelerationIncludingGravity属性，根据重力加速度变化判断手机是否剧烈摇动
2. 手机摇动条件成立时，css3实现红包展开动画，显示出canvas绘制的刮刮卡遮罩层
3. 刮刮卡此前已经随机生成了底部的金额，需要在canvas遮罩层上绑定鼠标事件，获取鼠标移动轨迹，清除该轨迹上的遮罩（只需）。
4. 每生成一次轨迹，获取所有像素值，检测是否已经清除了70%的遮罩，如此便清除所有遮罩，直接显示金额。



## 识别手机摇一摇

首先我们可以设置监听devicemotion事件的函数，事件对象中会包含一个重力加速度accelerationIncludingGravity属性，其包含x、y、z轴三个方向。利用这个属性就可以判断手机摇动。

那么如何判断有效的手机摇动？

这里利用了一个公式：

```
speed = Math.abs(x + y + z - this.last_x - this.last_y - this.last_z) / diffTime * 10000;
```

主要是获取一段时间内三个方向重力加速度差值和与时间的比值，再乘一个权重。这个方法不是十分准确，但可以作为一个简单的判断方式。

因为无需时时检测计算该值，因此增加一个判断条件，超过300ms判断一次：

`curTime - this.last_update > 300`

整个方法如下：

	function(e) { 
	    var acceleration = e.accelerationIncludingGravity;
	    var curTime = new Date().getTime();  //获取当前时间
	    var diffTime = curTime - this.last_update;
	    // 时间间隔300以上再判断
	    if(diffTime > 300) {
	    	// 根据上述公式计算结果
	        var x = acceleration.x;
	        var y = acceleration.y;
	        var z = acceleration.z;
	        var speed = Math.abs(x + y + z - this.last_x - this.last_y - this.last_z) / diffTime * 10000;
	        // 这里的SHAKE_THRESHOLD为800，也可替换为其他值，变化检测范围
	        if(speed > SHAKE_THRESHOLD) {  
	            // 识别到摇一摇之后的处理
	        }
	        // 更新时间和重力加速度
	        this.last_update = curTime;
	        this.last_x = x;
	        this.last_y = y;
	        this.last_z = z;
	    }
	}
## css3打开红包效果

![1](F:\wamp\www\demos\GitProject\AllDemos\css3-html5\yaoyiyao\1.PNG)![1](F:\wamp\www\demos\GitProject\AllDemos\css3-html5\yaoyiyao\2.PNG)

如图所示，红包以整个envelope容器的背景红色为最底层，依次向上是两行文字、canvas灰色遮罩、红色信封尾部、深红色的信封头部。其中头部还包含一个开字作为子元素，如下为html结构：

		<div class="enve-wrapper">
	        <h3>刮开查看</h3>
			<h2 id="money-text">20元</h2>
			<canvas id="cover" width=80 height=60></canvas>
			<div class="bottom"></div>
			<div class="top">
				<div class="open">
					<span>开</span>
				</div>
			</div>
		</div>
css部分只需注意envelope容器要设置overflow：hidden，使得动画移出的部分隐藏。

识别到手机摇动后，信封头部向上移出容器，信封尾部向下移出容器。实现开启信封的效果。

## canvas实现刮刮乐

###canvas的globalCompositeOperation属性

刮刮乐主要运用了canvas的globalCompositeOperation属性，它规定了上一路径与下一路径的堆叠效果。

一共有如下几种不同的取值：

![https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1517399455000&di=d8a487c91fc9dfb773d735c8cfdf6158&imgtype=0&src=http%3A%2F%2Fimg.askh5.com%2Fuploads%2Farticle%2Fd7%2Fe4%2F47%2Fd7e447f4f80bc0212e21ad4701c36d44.jpg]()

蓝色部分为原始的路径，红色为之后的路径。

我们需要用到的是 'destination-out'，这样便可让灰色遮罩作为原始路径，之后每绘制一笔，都将清除该路径上的灰色。

那么接下来只要识别鼠标按下路径。

###获取鼠标路径

我们知道鼠标事件只能获取到相对于页面的位置，所以需要利用getBoundingClientRect()方法获取canvas相对于页面的位置，做差就是鼠标相对于canvas的坐标点。

鼠标按下后，存下此刻位置作为起始点，鼠标移动后获取的位置为终点，这是第一次绘制路径，之后的每次都是以上一次的终点最为起始点，这一次鼠标移动后获取的位置为终点。所以需要用变量lastPointX、lastPointY记录每个终点，以便下次绘图。

###事件绑定和解除

需要注意事件绑定和解绑时机以及事件绑定的对象：mousemove、mouseup需要在mousedown之后绑定，且mouseup需要绑定在document上，否则在canvas外放下鼠标会有bug。

## 即时对象初始化模式来实现所有功能

即时对象初始化模式，就是将所有的**功能和属性定义在一个对象字面字面量**中，再**立即调用其中的一个方法**，完成初始化操作。形式如下：

	({
		// 定义一些属性
	    SHAKE_THRESHOLD : 800,
	    last_update : 0,
	    last_x : 0, 
	    last_y : 0, 
	    last_z : 0,
	    // 定义一些方法
	    init : function () {
	       // 函数实现
	    }
	}.init())
这个模式的好处是结构化，并且不污染全局变量。但是由于这个demo中需要绑定和解除事件，需要注意调用方法时的**this指向**，实现时会有一些坑。

mousedown的事件绑定：这里我们把公共属性定义到了对象字面量中，为了访问到这些属性，事件处理函数也需要定义为对象字面量中，才能通过this访问，并且还需要利用**bind方法确保是该对象字面量调用事件处理函数**。

mousemove、mouseup的事件绑定：这里直接将函数定义在mousedownHandle中，以便可以直接**通过闭包来访问对象字面量**，也方便绑定事件和解绑时能访问到对应的事件处理函数。

	init: function () {
	    // 一些处理
	    // 注意bind改变this指向
	    this.canvas.addEventListener('mousedown', this.mousedownHandle.bind(this), false);
	},
	mousedownHandle: function (e) {
	    // ...
	    var that = this;  //存储this，以便子函数可以访问
	    function mousemoveHandle(e) {
	        that.nowPointX = e.clientX - left;
	        that.nowPointY = e.clientY - top;
	        // ...
	    }
	    function mouseupHandle() {
	        that.canvas.removeEventListener('mousemove', mousemoveHandle);
	        document.removeEventListener('mouseup', mouseupHandle);
	        that.clearAll();
	    }
	    this.canvas.addEventListener('mousemove', mousemoveHandle, false);
	    document.addEventListener('mouseup', mouseupHandle, false);
	},
如果依然把mousemove、mouseup的事件处理函数像mousedown一样定义为对象字面量属性，那么依旧需要bind方法改变this指向，这时将**无法解除事件绑定**。因为**bind返回了一个新的匿名函数，无法再获取到该匿名函数的引用**！