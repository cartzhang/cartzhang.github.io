

## 一、手柄渲染总体流程

这篇来讨论和分析一下，steam VR插件在unity中怎么渲染手柄的模型。
很多人可能觉得vive 手柄很神秘，在steam vr导入到unity中后，没有看到任何手柄或头盔的模型，手柄确确实实被渲染到场景中了，这是怎么回事？

这篇文章就来抽丝剥茧的来说明，揭开其面纱。
渲染一般的流程，就是模型读取然后加载，显示，这么多。

根据代码流程绘制了简单的导图。

![image](https://github.com/cartzhang/cartzhang.github.io/raw/master/images/vivecontroller/ViveControllerRender/ViveController%E6%89%8B%E6%9F%84%E6%B8%B2%E6%9F%93.png)

图 手柄渲染导图

说明下：

当前使用的是steam 1.1.1版本插件，而当前最新的插件版本为1.2.1版本，里面代码openvr API做了不少细节的修改，这个修改有一些变化，最明显的的就是steam 中event事件调用不在使用字符串来处理，这个很多代码就可能需要修改。

## 二、steam VR 渲染

首先来看看，在unity中检视板内的排列

![image](https://github.com/cartzhang/cartzhang.github.io/raw/master/images/vivecontroller/ViveControllerRender/0.png)

图0

左右手柄节点，然后是相机节点。左右手柄下有一个模型节点。
在不运行的情况下，是这样的。

![image](https://github.com/cartzhang/cartzhang.github.io/raw/master/images/vivecontroller/ViveControllerRender/Render.PNG)

只有这一个rendermodel这个脚本。

先给出个总的手柄渲染代码流程图。

![image](https://github.com/cartzhang/cartzhang.github.io/raw/master/images/vivecontroller/ViveControllerRender/RenderFlow.PNG)

这个图我自己分析画的，不是很专业，但是足以说明问题。

下面就会逐个分析来破解这个渲染流程，然后我们再通过代码来自己来修改，把手柄其他模型来看看。

## 三、代码分析

首先代码分析，我们可以看到有两个关键代码文件，一个是SteamVR_ContollterManager.cs,一个是SteamVR_RenderModel.cs。他们分别在[CamearRig]和LeftiControl和RightController的 Model上。如下图。

![image](https://github.com/cartzhang/cartzhang.github.io/raw/master/images/vivecontroller/ViveControllerRender/1.png)

图1

![image](https://github.com/cartzhang/cartzhang.github.io/raw/master/images/vivecontroller/ViveControllerRender/Render.PNG)

图 Render

### 1.SteamVR_ContollterManager.cs文件分析

我从SteamVR_ContollterManager来看看。

第一，在OnEnable来注册硬件连接，也就是手柄连接上的事件。

代码中写了注释可以看看，大概也说明了怎么回事。

```
void OnEnable()
	{
		for (int i = 0; i < objects.Length; i++)
		{
			var obj = objects[i];
			if (obj != null)
				obj.SetActive(false);
		}

		OnTrackedDeviceRoleChanged();

		for (int i = 0; i < SteamVR.connected.Length; i++)
			if (SteamVR.connected[i])
				OnDeviceConnected(i, true);

		SteamVR_Utils.Event.Listen("input_focus", OnInputFocus);
		// 这个用来注册硬件连接事件。事件为steam 自己写的事件。
		SteamVR_Utils.Event.Listen("device_connected", OnDeviceConnected);
		// 这个用来注册设备发生了互换事件。
		SteamVR_Utils.Event.Listen("TrackedDeviceRoleChanged", OnTrackedDeviceRoleChanged);
    }
```
第二，在注册事件内，产生模型或更新模型。
在

```
private void OnDeviceConnected(params object[] args)
```

这里只需要看Refresh()就可以了。

第三，Refresh里面的调用了SetTrackedDeviceIndex，而当时被Index与之前不同的时候，会广播"SetDeviceIndex"，SteamVR_RenderModel中最后一个函数就是

```
public void SetDeviceIndex(int index)
	{
		this.index = (SteamVR_TrackedObject.EIndex)index;
		modelOverride = "";

		if (enabled)
		{
			UpdateModel();
		}
	}
```

这样代码中就controllerManager文件到了RenderModel文件。

### 2.SteamVR_RenderModel.cs文件分析

进入这个文件里，代码执行的一件事情就是创建模型或更新模式。

第一，UpdateModel(),
打开这个文件你会发现好几个地方都会调用这个函数。
分别为OnModelSkinSettingsHaveChanged()，OnDeviceConnected()，OnEnable(),
再有就是刚才的SetDeviceIndex(int index)，一般在开始的话，就是从这里开始调用创建的。

第二，获取模型名称。
怎么获取模型名称呢，这个名称在那里呢？
就是在steam安装目录里。

![image](https://github.com/cartzhang/cartzhang.github.io/raw/master/images/vivecontroller/ViveControllerRender/modelpath.PNG)

==图片steam 模型地址。==

代码为：

```
// 由当前硬件设备的index，来获取设备属性，在设备属性里面有对象的模型名称。
var error = ETrackedPropertyError.TrackedProp_Success;
		var capacity = system.GetStringTrackedDeviceProperty((uint)index, ETrackedDeviceProperty.Prop_RenderModelName_String, null, 0, ref error);
		if (capacity <= 1)
		{
			Debug.LogError("Failed to get render model name for tracked object " + index);
			return;
		}

		var buffer = new System.Text.StringBuilder((int)capacity);
		system.GetStringTrackedDeviceProperty((uint)index, ETrackedDeviceProperty.Prop_RenderModelName_String, buffer, capacity, ref error);
```

再说一遍，**就是由当前硬件设备的index，来获取设备属性，在设备属性里面有对象的模型名称，也可以说是由硬件里面有标识码，然后根据标识码找到对应的模型名称**。这是也做硬件的朋友学过vive 开源lighthouse后给我讲述的，就这几行代码中得到验证。

第三，有模型名称了，就加载模型。

获取模型名称后，获取模型有多少个组件，然后获取组件模型名称。一个手柄模型有多个模型来组成的。
可以看看这里，常规的就是用这个来组合的，每个fbx文件就是其中的一个组件。


![image](https://github.com/cartzhang/cartzhang.github.io/raw/master/images/vivecontroller/ViveControllerRender/modelComponent.PNG)
==图 模型组件==

第四，异步加载完模型，加载材质纹理。


第五，SetModel(),给检视板中模型添加组件MeshFilter和MeshRender。




```
gameObject.AddComponent<MeshFilter>().mesh = model.mesh;
				gameObject.AddComponent<MeshRenderer>().sharedMaterial = model.material;
```
当然,没有组件添加组件，然后把mesh和材质赋值上去。

到这里vive手柄模型渲染的过程算是全部分析完毕。

**最重要的还是文章开始的两张图，一张是思维导图，一张是渲染流程图。**

## 四、修改模型示例

修改手柄渲染模型，方法有多中，把新的模型放到场景节点下，直接隐藏显示也是一种方法。这里说几种常用方法。

### 1.修改纹理

代码直接修改材质，这个需要重新材质texture。这个方法有什么问题呢，就是新版1.2.1的steamvr的插件中，不再是这个样子，而改用了宏，没有使用字符串来做事件监听的方法论。


```
void OnEnable ()
    {
        //Subscribe to the event that is called by SteamVR_RenderModel, when the controller mesh + texture, has been loaded completely.
        SteamVR_Utils.Event.Listen("render_model_loaded", OnControllerLoaded);
    }
```

所以,旧版本steam  vr 1.1.1版本的朋友还可以继续使用。

参考的代码：http://pastebin.com/ihcxi39x

这个不太推荐，当然若对你有用，你也记得点赞。

### 2 修改加载模型

在rendermodel代码中修改,模型名称，
![image](https://github.com/cartzhang/cartzhang.github.io/raw/master/images/vivecontroller/ViveControllerRender/fireEx.PNG)

这就搞定了。

看结果，我的手柄成灭火器了：

![image](https://github.com/cartzhang/cartzhang.github.io/raw/master/images/vivecontroller/ViveControllerRender/fireExtringer.PNG)

图 灭火器

当然还弄了类似把关二爷的青龙偃月刀，您瞧一眼，手柄变成了大刀。

![image](https://github.com/cartzhang/cartzhang.github.io/raw/master/images/vivecontroller/ViveControllerRender/modeldao.PNG)

图 大刀

只有做好了模型，这样是不是省心省力啊。神奇吧。

### 3. 根据硬件编码，直接修改vive手柄的原始模式匹配。

这个是最根本上来做的，不需要unity或其他引擎软件，直接修改了手柄硬件 匹配码，这个得从lighthouse付费开源说起了，现在VR行业中百花齐放的手柄硬件应该说绝大多数都是这类产品了。

不过，我只知道，没有付费学习过，还请高手赐教。


## 五、参考

【1】http://pastebin.com/ihcxi39x

【2】https://steamcommunity.com/app/358720/discussions/0/385428943467404154/


csdn blog: http://blog.csdn.net/cartzhang/article/details/61913354

标签：steam,VR,手柄,Unity,模型修改