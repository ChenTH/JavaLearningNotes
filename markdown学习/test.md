# 文档撰写
## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
###### 六级标题

> 引用了别人的文章
[链接](http://www.baidu.com)

This is an H1
=============
This is an H2
-------------

+ Red
+ Blue
+ Green
* Hello
* Hi
- Bird
- McHale
- Parish
1. what
2. happened

<ol>
<li>Bird</li>
<li>McHale</li>
<li>Parish</li>
</ol>


    class Student {
        private String id;
    }

***
*  * *
---
-   -  -


`create database hero;`

```SQL
create database hero;
```

```python
@requires_authorization
class SomeClass:
    pass

if __name__ == '__main__':
    # A comment
    print 'hello world'
```
```javascript
    function fun(){
         echo "这是一句非常牛逼的代码";
    }
    fun();
```

| 项目        | 价格   |  数量  |
| --------   | -----:  | :----:  |
| 计算机     | $1600 |   5     |
| 手机        |   $12   |   12   |
| 管线        |    $1    |  234  |
    第二行分割表头和内容。
    文字默认居左
    -两边加：表示文字居中
    -右边加：表示文字居右
    注：原生的语法两边都要用 | 包起来。此处省略



**这是加粗的文字**

*这是倾斜的文字*

***这是斜体加粗的文字***

~~这是加删除线的文字~~

`Ctrl+Alt+M`

>这是引用的内容
>>这是引用的内容
>>>>>>>>>>这是引用的内容


![jianshu](https://cdn2.jianshu.io/assets/web/nav-logo-4c7bbafe27adc892f3046e6978459bac.png "简书")

```flow
st=>start: 开始
cond=>condition: Yes or No?
op=>operation: My Operation
cond2=>condition: cond2
op2=>operation: result
e=>end
st->op->cond
op2->e
cond(yes)->op2
cond(no)->cond2
cond2(yes)->op2
cond2(no)->op
```

$x+y=z$
$f(x)=sin(x)+123$

文字的转义：
 特殊符号为 &#n; 其中n为特殊符号的ASCII码
 例如小于号的转义应为 &#60;