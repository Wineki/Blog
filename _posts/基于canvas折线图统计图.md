title: 基于canvas折线图统计图
date: 2016-04-18 19:02:36
tags:
---
这段时间一直在做人口统计折线图的需求，终于成功的上线了，打算把技术点总结一下，分享粗来。
    首先来看看这个磨人的小妖精长成什么样子，<a href="https://www.so.com/s?ie=utf-8&shb=1&src=360sou_newhome&q=%E4%BA%BA%E5%8F%A3" target="_blank">pc端的样纸</a>，<a href="http://m.so.com/s?q=%E4%BA%BA%E5%8F%A3&src=msearch_test&srcg=home_next" target="_blank">移动端的样纸</a>。pc和移动是两种不同的交互模式，绘制的过程中也有一些细节不太一样，在下面的介绍中会有提到，你要坚持读下去哦，来，我们再来张截图留个念~<img src="/img/pada.gif" width="30" height="30" style="display: inline-block" />
    pc:
    <img src="/img/chart1.png" alt="" width="400" height="200">
    移动：
    <img src="/img/chart2.png" alt="" width="400" height="200">
恩，炫耀够了，现在开始讲干货。
数据是根据数据端的同学爬数据动态统计出来的，我需要做的就是讲这些统计出来的数据画成统计图，展现出来。
这里主要介绍一下几点：
<li>绘制前对数据的处理</li>
<li>根据数据绘制折线图的过程</li>
<li>canvas在retina屏幕下显示效果的兼容</li>
<li>对IE6等低版本浏览器的兼容</li>
首先我们来介绍对数据的处理，由于代码是写在php模板中的，所以所有的数据会以smarty变量的方式传到前端模板中，而canvas进行渲染的数据一定要放在js变量中，所以就需要将smarty中的数据放到js变量中。 <br/>  
<pre>var data = $.parseJSON('{ %json_encode($result.display)% }');</pre>
 
解释一下，首先使用PHP函数json_encode对数据进行json编码，然后我们使用jquery中parseJSON方法对编码后的数据进行解码。这里parseJSON传入的参数要求比较高，稍不符合标准就gg了（官方说法：抛出异常）：<br/>
<li>{test: 1} (test 没有使用双引号包裹).</li>
<li>{'test': 1} ('test' 用了单引号而不是双引号包裹).</li>
<li>"{test: 1}" (test 没有使用双引号包裹).</li>
<li>"{'test': 1}" ('test' 用了单引号而不是双引号包裹).</li>
<li>"'test'" ('test' 用单引号代替双引号).</li>
<li>".1" (number 必须以数字开头; "0.1" 将是有效的).</li>
<li>"undefined" (undefined 不能表示一个 JSON 字符串; 然而null,可以).</li>
<li>"NaN" (NaN 不能表示一个 JSON 字符串; 用Infinity直接表示无限也是不允许的).</li>
现在数据已经缓存到js中了，接下来就是对数据的处理，数据处理共分为两部分，一部分是对横纵坐标的处理，另一部分是对折线图中的点的坐标的处理。
先说对横纵坐标的处理，由于伟大的PM是个处女座的美男子，对于数据处理要求十万分复杂，在若干次的愉快的争论和商讨后，我们决定将纵坐标展示逻辑：第一个坐标和最后一个坐标展示为数据中的最小值-误差值和最大值+误差值，误差值：数据的（最大值-最小值）*0.05再四舍五入，横坐标排布方案：将除前几个数平均分，剩余的所有都堆积在最后一段。是不是没太听明白，没事，我当时也各种懵圈，上个图片你自己算算就明白了。这是横坐标，你计算一项相邻坐标的差，就明白我的意思啦！
横坐标：<img src="/img/chart3.png" alt="" width="400" height="200" style="display:inline-block">
纵坐标：<img src="/img/chart4.png" alt="" width="100" height="200" style="display:inline-block">
来看看代码处理逻辑：

    /*Y轴上下误差*/
    var errVal = Math.round((yVal[0] - yVal[yVal.length-1]) *0.05);
    var minY = yVal[yVal.length-1] - errVal;
    minY = minY > 0 ? minY : yVal[yVal.length-1]*0.95;
    var maxY = yVal[0] + errVal;
    if(data.cpi_year !== undefined){
        /*根据具体业务逻辑，此种情况下，采用进一法处理数据*/
        var ySpacing = ((maxY - minY)/3).toFixed(1);
    }else{
        var ySpacing = Math.ceil((maxY - minY)/3);
    }

<br/>
当然实际的业务逻辑比现在说的还会复杂一些，这里就不一一介绍了，我们来看看生成横纵坐标的方法：
<br/>

    /*生成坐标轴*/
    function creatCoordinate(param,isX){
        var _this = this;
        var Carr = new Array();
        if(isX){
            for(var i=0;i<4;i++){
                $(param.parentNode,chart).append('<span class="'+param.className+' js-mh-xChart'+i+'">'+param.val+param.unit+'</span>');
                /*计算当前值与最小值的差值映射在坐标图中当前点距离最左边点的距离*/
                var left = parseInt(((param.val - param.minVal) * mapWidth /(param.maxVal - param.minVal)) - $('.js-mh-xChart'+i).width()/2);
                $('.js-mh-xChart'+i)[0].style.left=left+"px";
                Carr.push(param.val);
                param.val += param.spacing;
            }
            /*生成最后一个点*/
            $(param.parentNode,chart).append('<span class="'+param.className+' js-mh-xChart'+i+'">'+param.maxVal+param.unit+'</span>');
            var left = parseInt(((param.maxVal - param.minVal) * mapWidth /(param.maxVal - param.minVal)) - $('.js-mh-xChart'+i).width()/2);
            $('.js-mh-xChart'+i)[0].style.left=left+"px";
        }else{
            for(var i=0;i<3;i++){
                var valY = dataYround(param.val);
                $(param.parentNode,chart).prepend('<p class='+param.className+'>'+valY.y+valY.unit+param.unit+'</p>');
                Carr.push(param.val);
                param.val += +param.spacing;
                param.val = +(param.val).toFixed(2);
            }
            /*生成最后一个点*/
            var valY = dataYround(param.maxVal);
            $(param.parentNode,chart).prepend('<p class='+param.className+'>'+valY.y+valY.unit+param.unit+'</p>');
        }
        Carr.push(param.maxVal);
        return Carr;
    }   
解释一下几个具体的点：isX为true为x轴处理逻辑，false为y轴处理逻辑。for循环将所有的点渲染出来，最后将最后一个点的位置单独绘制出来，单独绘制的原因之前说业务逻辑的时候有提到。这里需要强调一点，在x轴处理的时候需要一边渲染数据一边计算当前值与最小值得差值映射在坐标图中距离最左边点的距离，同时需要减去当前点的文案长度的1/2。
到目前为止横纵坐标已经处理完成了，现在要开始画折线图啦！canvas神奇闪亮登场！！欢呼欢呼！！我终于写完四分之一了！<img src="/img/pada1.gif" width="30" height="30" style="display: inline-block" />
canvas绘制首先新建一个chart对象。具体结构如下：

        /*绘制表格*/
        function Chart( config ){
            var _this = this;
            _this.ele = {
                
            };
            _this.data = {
                
            };
        }

以上是画图所需要的所有参数的，现在我们基于这些定义好的参数开始绘制折线图。
首先需要initCanvas画布

    initCanvas : function(className){
                var _this = this;
                var canvasEle = document.createElement('canvas');
                var width = +_this.data.chartWidth;
                var height = +_this.data.chartHeight;
                $(canvasEle).width(width)
                            .height(height)
                            .attr('width',width)
                            .attr('height',height)
                            .addClass(className);
                /*针对IE8以下低版本浏览器的处理*/
                if(_this.data.IE8_AND_LOWER){
                    canvasEle=window.G_vmlCanvasManager.initElement(canvasEle);
                }
                _this.ele.canvasWrap.append(canvasEle);
            }
传入包裹canvas的外层容器，动态创建canvas元素，并为canvas元素复制高度和宽度，这里需要注意的一点是：细心的你有么有发现我们在attr和style里面分别定义了两次宽度和高度，这个其实就是为了兼容retina屏幕2倍尺寸做的处理，style中的width和height是我们初始化时候的值，当你的屏幕是retina屏的时候，attr中的width和height就会处理成乘以2的值。if判断中对IE8以下的浏览器，无法获取getContext的方法，采用<a href="https://github.com/yinso/excanvas" target="_blank">excanvas</a>这个插件来兼容。
然后开始画背景
 
    drawChart: function(){
                var _this = this;
                var chartCtx = _this.ele.canvasChart.getContext('2d');
                /*中心平移，防止边缘反锯齿*/
                chartCtx.translate(0.5, 0.5);
                /*画背景网格*/
                _this.drawBackground(chartCtx);
                /*画数据*/
                _this.drawLine(chartCtx);
            }
    drawBackground: function(ctx){
                var _this = this;
                ctx.beginPath();
                ctx.strokeStyle = '#dcdcdc';
                ctx.lineWidth = 1.0;
                var lineSpace = _this.data.lineSpace;
                for(var i=0;i<_this.data.lineNum;i++){
                    var lineY = parseInt(i * lineSpace);
                    ctx.moveTo(0,lineY+_this.data.tOffset);
                    ctx.lineTo(_this.data.chartWidth, lineY+_this.data.tOffset);
                }
                ctx.stroke();
            }
lineNum为背景线的总个数，定义背景线的颜色，宽度，两线间隔，起点是（0，lineY+_this.data.tOffset）lineY为通过根据两线间的间隔计算的值，tOffset为距离顶部的位移量，终点是（_this.data.chartWidth, lineY+_this.data.tOffset）_this.data.chartWidth表示图表的宽度。效果图:<img src="/img/chart5.png" alt="" width="400" height="200">。
这里需要强调一下，<em>chartCtx.translate(0.5, 0.5);</em>这里的用意，是因为在绘制宽度为1的线条时，canvas是以线条宽度中间位置作为渲染中心像上下方向晕染，所以为了清晰成像，我们将画布平移二分之一宽度。
现在可以在这个背景的上面绘制折现图了，需要声明的是在绘制折线图之前，需要对数据处理，此处细节忽略一万字......
假如我们的数据已经处理好了哦，现在开始画图

    drawLine: function(ctx){
                /*绘制折线图*/
                var _this = this;
                /*getposition方法为忽略一万字的逻辑*/
                _this.getPosition(_this.data.points);
                ctx.strokeStyle = '#19b955';
                ctx.fillStyle = '#19b955';
                ctx.lineWidth = 2.0;
                ctx.beginPath();
                for(var i=0; i<_this.data.pointsPosition.length;i++){
                    if(i + 1 === _this.data.pointsPosition.length){
                        break;
                    }
                    ctx.moveTo(_this.data.pointsPosition[i][0],_this.data.pointsPosition[i][1]);
                    ctx.lineTo(_this.data.pointsPosition[i+1][0],_this.data.pointsPosition[i+1][1])
                }
                ctx.stroke();
            }
其实做法和画背景图的线差不多，_ this.data.pointsPosition.length表示的是要绘制的点的长度，确定好折线图的颜色和宽度，将整个折线图分割成为若干个小线段，每个小线段的起点是当前点，终点是下一个点。
这时候折线图就画好了，看看效果：
<img src="/img/chart6.png" alt="" width="400" height="200">
现在到最后一步了，我们来绘制hover效果。处理思路就是：你需要获取当前鼠标hover在canvas中的x轴的位置，然后根据hover点的x坐标获取对应点的实际数据，同时利用当前的x坐标画出hover的虚线，再利用对应的实际数据画出hover的圆点。

        /*画鼠标hover圆点以及竖线*/
        drawArcPoint: function(ctx,pointData){
                var _this = this;
                /*画出横坐标的动态值*/
                _this.ele.tipsTextWrap.text(pointData.x);
                _this.ele.tipsTextWrap[0].style.left = _this.data.arcPos.x - _this.ele.tipsTextWrap.width()/2 +'px';
                /*这里要实时跟踪鼠标位置，所以在重新绘制之前应该清除之前绘制的圆点和竖线*/
                ctx.clearRect(0,0,_this.data.chartWidth,_this.data.chartHeight);
                /*开始绘制*/
                ctx.beginPath();
                /*定义线的颜色*/
                ctx.strokeStyle = '#dcdcdc';
                /*定义线的宽度*/
                ctx.lineWidth = 1.0;
                /*起始点*/
                ctx.moveTo(_this.data.arcPos.x,0);
                /*终点*/
                ctx.lineTo(_this.data.arcPos.x,_this.data.mapHeight + _this.data.tOffset);
                ctx.stroke();
                if(pointData.y !== undefined){
                    /*绘制圆点*/
                    ctx.beginPath();
                    ctx.arc(_this.data.arcPos.x,_this.data.arcPos.y,4.5,0,Math.PI*2,true);
                    ctx.fillStyle = "#19b955";
                    ctx.fill();
                }
            }
当然还处理一下边界的问题，这里就不讲这个细节了~至此折线图就画完了，以上讲述的都是整个代码中的部分关键点，还有一些具体的操作并没有详细来讲，我写了两个小demo（<a href="/demo/chart.html" target="_blank">pc端</a>，<a href="/demo/m-chart.html" target="_blank">移动端</a>），包含移动和PC两部分，如果你感情去的话可以下载源码自己试试哈~
恩，现在我们来说说，对retina屏幕的兼容问题，像你了解的那样，retina是DPR=2[DPR(devicePixelRatio) = 设备像素 / CSS像素],来看下面这个图：<img src="/img/chart7.png" width="400" height="200" alt="">,所以正常绘制的话，就会导致retina屏幕呈现的效果有些模糊，所以我们用了一个兼容的插件<a href="https://github.com/jondavidjohn/hidpi-canvas-polyfill" target="_blank">hidpi-canvas-polyfill</a>处理原理呢就是：获取window.devicePixelRatio的值，将canvas所用到的绘制数据都乘以对应的DPR的值，然后就可以清晰地成像了。
最后的最后，来介绍一下IE8以下老版本浏览器兼容canvas神器，前面提到过得excanvas.js,使用方式：

          canvasEle=window.G_vmlCanvasManager.initElement(canvasEle);
原理是由于老版本浏览器无法获取getContext的方法，通常情况下，脚本会报错，而excanvas完美的将canvas转换成为IE喜欢的VML(Vector Markup Language)格式进行绘制。
好了，写完了，累shi宝宝了......<img src="/img/pada3.gif" width="30" height="30" style="display: inline-block" />