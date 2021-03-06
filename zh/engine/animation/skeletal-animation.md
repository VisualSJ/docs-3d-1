
# 骨骼动画

骨骼动画是一种常见但类型特殊的动画，我们针对性地做了很多底层优化。
目前我们的骨骼动画运行时的完整处理流程如下：
* 骨骼动画片段（SkeletalAnimationClip）资源会在加载时把动画数据自动预采样、转换到世界空间；
* 当一个骨骼动画组件（SkeletalAnimationComponent）接到播放指令（play）后，会把将要播放的动画片段通知给所有自己控制的蒙皮模型组件（SkinningModelComponent），
  在这里直接将完整的动画片段数据以纹理形式上传 GPU；
* 动画组件每帧驱动这些蒙皮模型组件，传递当前播放到的帧数（frame ID）；
* 蒙皮模型组件每帧将 frame ID 以 uniform 形式上传 GPU，在 GPU 完成这一帧的蒙皮。

## 蒙皮算法

我们内置提供两种常见标准蒙皮算法，它们性能相近，只对最终表现有影响：

1. LBS（线性混合蒙皮）：骨骼信息以 3x4 矩阵形式存储，直接对矩阵线性插值实现蒙皮，有体积损失等典型已知问题；
3. DQS（双四元数蒙皮）：骨骼信息以双四元数形式插值，对不含有缩放变换的骨骼动画效果更精确自然，但出于性能考虑，对所有缩放动画有近似简化处理。

引擎默认使用 LBS，可以通过修改引擎 skeletal-animation-utils.ts 的 `updateJointData` 函数引用与 cc-skinning.chunk 中的头文件引用来切换蒙皮算法；
推荐对蒙皮动画质量有较高追求的项目可以尝试启用 DQS，但因 GLSL 400 之前都没有 `fma` 指令，如 `cross` 等操作在某些 GPU 上无法绕过浮点抵消问题，误差较大，可能引入部分可见瑕疵。

## 纹理格式

根据运行平台是否支持浮点纹理，会对应使用 RGBA32F 或 RGBA8 的 fallback，这层用户不必关心，不对最终表现有影响，只是低端平台最后的保底策略。

## 挂点系统

如果需要将某些外部节点挂到指定的骨骼关节上，需要使用骨骼动画组件的挂点（Socket）系统：
* 在要对接的骨骼动画组件下新建一个子节点（直属 parent 应为动画组件所在节点）；
* 在骨骼动画组件的 sockets 属性中添加一个数组元素，path 从下拉列表中选择要挂接的那根骨骼的路径，target 指定为刚刚创建的子节点；
* 这个子节点就成为目标挂点了，可以把任何外部节点放到这个子节点下，都会跟随指定骨骼变换了。

FBX 或 glTF 资源内的挂点模型会自动对接挂点系统，无需任何手动操作。

## 关于不播放动画时的默认姿势
每个模型资源在导入后都有一个固定的 “默认姿势”，与绑骨骼时确定的 “绑定姿势” 不同，这个姿势一般是美术在 DCC 工具中选择导出时，场景中当前这一帧的姿势。
而目前的运行时骨骼动画框架，出于性能考量，除标准的以 animation clip 为基本单位的动画外，只支持显示 “绑定姿势”，暂时不支持显示 “默认姿势”；
这主要体现在编辑器中，模型资源拖入场景后，在不播任何动画片段时，显示效果会和第三方工具略有差异，一般不影响使用；
但当绑定姿势和动画片段动作差异较大，甚至坐标轴也完全不同时，在调整模型时，请一定以播放动画片段时的状态为准。

## 数据合批

目前底层上传 GPU 的骨骼纹理已做到全局自动合批复用，上层数据目前可以通过使用 批量蒙皮模型组件（BatchedSkinningModelComponent）将同一个骨骼动画组件控制的所有子蒙皮模型合并：

![](batched-skinning-model-component.png)

合批版 effect 书写相对复杂一点，但基本可以基于子材质使用的普通 effect，加入一些相对直接的预处理和接口改动即可，编辑器内的内置资源里 (util/batched-unlit) 提供了一个合批版 builtin-unlit，可以参考。
