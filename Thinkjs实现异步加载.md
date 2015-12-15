title: Thinkjs实现异步加载
date: 2015-12-14 17:23:10
tags:
---
今天来总结一下关于使用Thinkjs来实现页面异步加载的原理和做法。
我从前端讲起，首先前端模板中包含一下几部分：


| View          | Controller    | Model  |
| ------------- |:-------------:| -----:|
| 筛选框（filter）用于筛选内容，位置在母模版| 母模版对应的Action | 调用数据库 |
| 异步加载内容（content），位置在子模板| ajax判断后的处理 | 调用数据库  |

下面根据代码具体解释一下：
母模版的view层demo：


    /*这里是一个filter*/
    <div class="filter">
    	<dl class="row">
    		<dt>大小</dt>
    		<dd>
    			<a href="#">全部</a>
    			<a href="#">大</a>
    			<a href="#">中</a>
    			<a href="#">小</a>
    		</dd>
    	</dl>
    </div>
    /*这里是一个content*/
    <div class="content">
        <%include ajax/tplselectajax.html%>     /*这里include进来一个子模板，将来异步刷新时就只刷新子模板数据*/
    </div>
    
子模板的view层demo:
    可以看到从后端传来一个tpllist的数据字段，相当于是一个数组，我们把它循环输出出来，同时一会异步刷新的时候我们也只是刷新这个子模板的数据，而不会改变上面的母模板的数据。

    <div class="cont-wrap">
        <ul>
            <%tpllist.forEach(function(item){%>
                <li>
                    <p><%=item.desc%></p>
                    <p><%=item.name%></p>
                </li>
            <%})%>
        </ul>
    </div>
 

Controller层的处理:

    tplselectAction: function(){
        var self = this;
        /*获取get请求的参数，在这个demo中其实就是筛选条件*/
        var data = this.get();  
        var pg = data.pg;
        /*判断时候是ajax请求，如果是则结果返回true*/
        var isAjax = self.isAjax();   
        /*子模板渲染路径，注意此处根目录是Home*/
        var tpl = 'Home/ajax/build_tplselectajax.html';         
        if(!isAjax){
            /*如果不是ajax请求，也就是浏览器首次渲染数据时，默认选择展示全部数据*/
            data = { cateId: '全部'}
        }
        /*调用数据库获取数据，调用ajaxdata方法*/
        D('Template').ajaxdata(data,pg).then(function(data){
            var list = shiftObject(data,'data');  /*将数据库data字段整体存入list*/
            self.assign('tpllist',list);  
            self.assign('isAjax',isAjax);
            if(isAjax){
                /*如果是ajax模板，则利用fetch这个方法对模板进行重新渲染，并返回相应json格式的状态值*/
                if(list.length !== 0){
                    self.fetch(tpl).then(function(content){
                       self.jsonp({status:"success",cont:content});
                    });
                }else{
                    self.jsonp({status:"fail",cont:'暂时木有相应的模板，搜搜别的试试吧！'});
                }
            }else{
                /*首次加载页面则直接展示*/
                self.display();
            }
        });
    }
    
Model层的处理demo：
    
    ajaxdata: function(data){
        var self = this;
        if(data.cateId = '全部'){
            return self.order({'id':'desc'}).select().then(function(data){
                return data;
            });
        }else{
            return self.where({'title':data.cateId}).order({'id':'desc'}).select().then(function(data){
                return data;    
            });
        }
    }
Model层的处理：
这里是项目源代码的查询:

    ajaxdata: function(data,pg){
            var self = this;
            var where = where || {};
            /*此处包含其他参数的复合查询，展示出来，在demo中暂时不考虑*/
            /*var join = {
                templatecate: {
                    join: 'left',
                    as: 'c',
                    on: ['cateId','id']
                }
            };
            var pg = pg || (isNumber(where) ? where : null);
            var lastCond = {};
            if(!isEmpty(where)){
                for(var i in where){
                    var find = false;
                    for(var s in fields){
                        if(fields[s].indexOf(i) > -1){
                            lastCond[s + '.' + i] = where[i];
                            find = true;
                            break;
                        }
                    }
                    if(find)continue;
                    lastCond['t.' + i] = where[i];
                }
            }*/
            if(data.cateId == '全部'){
                var per = C('db_nums_per_page');
                /*在筛选框为‘全部’时，将数据中全部数据按条件查询出来，并且进行分页查询（分页查询此处我们暂时不讨论）*/
                return self.where({'endType':data.endType,'isForbidden':0}).order({'id': 'desc'}).page(pg).select().then(function(data){
                        return data;
                }).then(function(datalist){
                    /*将返回结果进行处理，并返回相应的参数*/
                    return self.field('count(id) as count').where({'endType':data.endType}).find().then(function(res){
                        return {
                            data: datalist,
                            count: res.count,
                            page: pg || 1,
                            num: per,
                            total: Math.ceil(res.count / per)
                        };
                    });
                }); 
            }else{
                /ajax请求是查询结果，按照cateId的值进行查询/
                var per = C('db_nums_per_page');
                return self.alias('t').field('t.*').join(join).where({'c.title':data.cateId,'t.endType':data.endType,'isForbidden':0}).order({'t.id': 'desc'}).page(pg).select().then(function(data){
                        return data;
                }).then(function(datalist){
                    return self.alias('t').field('count(t.id) as count').join(join).where({'c.title':data.cateId,'t.endType':data.endType}).find().then(function(res){
                        return {
                            data: datalist,
                            count: res.count,
                            page: pg || 1,
                            num: per,
                            total: Math.ceil(res.count / per)
                        };
                    });
                }); 
            }
            
        }