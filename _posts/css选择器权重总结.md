title: css选择器优先级权重分析
date: 2015-11-25 23:25:41
tags:
---
最近写项目发现了css的一些小问题总结出来，主要是关于css选择器权重相关的。
在浏览器中，我们在编写页面样式的时候通常都要考虑选择器的优先级，使得页面的样式不会被覆盖或覆盖其他样式。
总体来说优先级是这样的（此处从高到底排序）：
>!important  ->   行内样式style(此处包含写在html结构中的和用js动态加载上去的style)  ->  id选择器  ->  伪类  ->  class选择器  ->  元素选择器  ->  通用选择器( * )

这里我来对一些细节说明一下：
1.页面中能不要使用！important尽量避免使用，因为从严格意义上讲，！important与优先级是没有关系的，当使用!important定义模块的时候页面中再对其进行任何操作都无济于事了，改变了样式表本来的级联规则，难以调试。
2.行内样式这里的两方面距离说明：

    1.<p style="color:red" id="test"></p>

    2. document.getElementById('topnav').style.color = 'red'

3.关于伪类的优先级遵循：LVHA,从右向左优先级逐级增加.然而not作为伪类，却并不列入伪类优先级的评比中，也就是说当元素有重新定义的样式的时候not会被自动覆盖。举例说明：

    <div class="test-not">测试伪类not</div>
    <div class="not">123</div>
当定义样式为：

    .test-not:not(.not){
        color:red;
    }
效果如下图：
![](./img/1.png)
123为黑色，说明样式中not生效了。
当定义样式为：

    .not{
        color:red;
    }
    .test-not:not(.not){
        color:red;
    }
效果如下图：
![](./img/2.png)
123显示为红色，说明not样式被元素本身定义的样式覆盖了。
4.关于a标签的说明：

    a:link{
        color:green;
    }
    /*a:visited{
        color:yellow;
    }*/
    .test{
        color:red;
    }
    <a href="/" class="test-relative">相对目录</a>
当我们把a标签加上link伪类的时候，link的优先级就高于class="test",所以在首次载入页面的时候文字会显示绿色，当我们点击链接后，发现文字又变回了红色，这是因为链接点击后的状态为visited，而在样式中我们将visited的样式定义注释掉了，所以链接样式就会直接显示class="test"的样式了。
当我们将visited的样式定义去掉后，会发现颜色显示为黄色。
