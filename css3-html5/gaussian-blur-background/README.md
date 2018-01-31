# canvas实现图片高斯模糊

[TOC]

## 思路

1. 利用了canvas的ctx.getImageData方法，获取了图片中间的部分的数据
2. 再利用高斯模糊方法进行数据处理（大规模纯数据处理，因此在ui线程外部执行js代码）
3. 最后通过putImageData的方法绘制在另一个canvas上

## canvas的运用，注意属性canvas的width和css中width的区别

因为这个demo是希望图片以背景方式的形式显示，因此无法直接在canvas上设置width和height属性，于是使用css设置canvas的大小。

canvas绘图利用了drawImage方法，在此之前要设置canvas的width属性值与offsetWidth相等，height也是同理。**因为真正绘图是按照width属性大小绘制的，offsetWidth的大小可以理解为绘制后再进行缩放显示。**否则绘制图片时会导致比例扭曲，按照默认canvas的width=300，height=100绘制。

中间部分的canvas设置成了根据百分比来确定大小，这样可以保证调整窗口大小时，图像模糊的部分不变，就不用再次进行高斯模糊。

## js开启work.js新线程

高斯模糊处理是**大规模纯数据处理**，所以可以使用H5新支持的web workers，在ui线程外部执行计算任务。

实现方法是：

	// 主线程
	var worker = new Worker('./gaussWork.js');
	// 监听onmessage事件，在计算完成后的会调用
	worker.onmessage = function (e) {
	    // 得到新数据后的处理
	    console.log(e.data);
	}
	// 发送待处理的数据
	worker.postMessage(imgData);
	// gaussWork.js
	// 监听事件，处理数据
	onmessage = function (e) {
	  var imgData = e.data;
	  // 发送处理后的数据
	  postMessage(gaussBlur(imgData));
	}
其实就是通过postMessage方法和监听onMessage方法来进行数据传送和接收。

附上work的使用注意：

- worker文件必须和主文件满足同源策略
- worker只是window的子集，只能实现部分功能，不能获取到window,document等