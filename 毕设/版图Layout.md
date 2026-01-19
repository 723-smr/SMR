## 原理图

![](assets/版图Layout/file-20260119214235270.png)

然后一路点ok

## 和原理图connect：

![](assets/版图Layout/file-20260119214302610.png)

![](assets/版图Layout/file-20260119214323508.png)

出现：![](assets/版图Layout/file-20260119214350056.png)

## 按大写F变成：![](assets/版图Layout/file-20260119214410900.png)

> shift+f观看内部结构，ctrl+f返回宏观视角


## 换皮肤颜色：

![](assets/版图Layout/file-20260119214428406.png)
![](assets/版图Layout/file-20260119214435571.png)

CJR_DATA有一个drf文件:
![](assets/版图Layout/file-20260119214448311.png)

## 添加Guard Ring

![](assets/版图Layout/file-20260119214502466.png)

**pmos用n_FGR，nmos用p_FGR**
![](assets/版图Layout/file-20260119214514199.png)

Q完之后直接改名字为AGND/AVDD

  

>修改guardring长度：
>先点击这个：![](assets/版图Layout/file-20260119214537332.png)
鼠标放在线的末端，出现三个点后点击一下：![](assets/版图Layout/file-20260119214552480.png)
选中后按s键拖动即可

## 调整精度

按e
![](assets/版图Layout/file-20260119214623266.png)
## 连线
用p连线

## 量距离
用k去量，选中按delete就可以删除

## 改线宽
按p之后再F3

## 打孔
poly层是Gate，注入层在poly层下面，然后poly层上面依次是M1、M2、M3...
这里打孔的意思是从第几层打到第几层，比如说，A信号从poly到M1层，就打poly到M1的通孔，然后用线连起来：
![](assets/版图Layout/file-20260119214705629.png)

按o，出现如下：![](assets/版图Layout/file-20260119214727138.png)

![](assets/版图Layout/file-20260119214744462.png)
![](assets/版图Layout/file-20260119214800548.png)

#### `这里要特别注意一点！！！！！打通孔的时候，pmos的poly层要是polyp，NMOS是polyN，poly层不要选错！！！`


## DRC
![](assets/版图Layout/file-20260119214828613.png)

只需要选一下文件，剩下全部ok，然后直接Run DRC
![](assets/版图Layout/file-20260119214837086.png)

  

## LVS

>LVS = Layout Versus Schematic
👉 版图 vs 原理图一致性检查
![](assets/版图Layout/file-20260119214853138.png)

放到cjr的文件夹下：
![](assets/版图Layout/file-20260119214904496.png)

检查报错：
![](assets/版图Layout/file-20260119214912299.png)

LVS检查的是comparison results

## 添加nwell

![](assets/版图Layout/file-20260119214928124.png)

然后直接将pmos用矩形框住就行

原理图layout出来的pmos本身就有nwell，有时候需要再画一个nwell将几个pmos框在一起是因为这些pmos是并联的，拥有共同的VDD，所以需要再用一个nwell框在一起

  
## 小技巧：

按e键，然后选中Enable Dimming，就可以让未选中的区域变暗，这样方便看
#### `重点检查有没有大图时候接上的，但放大后就没接上的`

![](assets/版图Layout/file-20260119214944889.png)