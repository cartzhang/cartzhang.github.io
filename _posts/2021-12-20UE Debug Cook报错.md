
本文章由cartzhang编写，转载请注明出处。 所有权利保留。 
作者：cartzhang


一、COOK 问题

最近遇到一个UE5 的COOK问题。
项目版本从UE4.26.2升级到UE5 EA2 版本，升级完毕，项目打包一团乱。
首先遇到的就是COOK不过，各种报错。
由于我们的项目经过合并，原本有UE４开发的部分，也有部分人使用UE５　开发，造成了资源合并，删除等处理。
大多数都是资源的都是正常的，某些资源贴图或纹理就丢失了，材质函数变化了，等。

有一个离奇的问题，在于COOK过程中直接报错了，直接丢出一个堆栈。

![image](https://github.com/cartzhang/cartzhang.github.io/raw/master/images/UEDebugCook/cookdebug1.png)

是不是很尴尬
二、调试

直接有了解过，可以直接调试。今天就自己试试。

怎么操作呢。

1、首先，打开源码引擎,把引擎设置为启动项。

2、然后：

在属性中添加Debug 参数。

![image](https://github.com/cartzhang/cartzhang.github.io/raw/master/images/UEDebugCook/cookdebug.png)

3、运行即可。
我们这里报错直接指向某个UMG资源。在UE4升级UE5过程中，序列化的时候，指针数组越界了。
然后找到这个家伙就可以了。

大功告成！


三、参考
https://docs.unrealengine.com/4.27/en-US/SharingAndReleasing/Deployment/Cooking/
https://zhuanlan.zhihu.com/p/75168849

