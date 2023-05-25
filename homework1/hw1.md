# homework1

作业1，扩展GLTF loading。作业1提供了一个gltf显示的demo，只支持静态模型，以及颜色贴图。作业1需要在这个基础上进行升级。
#### 作业提交

按照代码框架的目录（方便助教检查和运行代码），把修改的文件打包成zip，或者用git patch的方式提交作业代码。

#### 作业要求
1. 作业要求的gltf文件已经上传到了data/buster_drone/busterDrone.gltf
2. 支持gltf的骨骼动画。
3. 支持gltf的PBR的材质，包括法线贴图。
4. 必须在homework1的基础上做修改，提交其他框架的代码算作不合格。
5. 进阶作业：增加一个Tone Mapping的后处理pass。增加GLTF的滤镜功能。tonemap选择ACES实现如下。这个实现必须通过额外增加一个renderpass的方式实现。
```c++
// tonemap 所使用的函数
float3 Tonemap_ACES(const float3 c) {
    // Narkowicz 2015, "ACES Filmic Tone Mapping Curve"
    // const float a = 2.51;
    // const float b = 0.03;
    // const float c = 2.43;
    // const float d = 0.59;
    // const float e = 0.14;
    // return saturate((x*(a*x+b))/(x*(c*x+d)+e));

    //ACES RRT/ODT curve fit courtesy of Stephen Hill
	float3 a = c * (c + 0.0245786) - 0.000090537;
	float3 b = c * (0.983729 * c + 0.4329510) + 0.238081;
	return a / b;
}
```

直接运行会不成功缺少GLTF模型。以及字体文件。根据[文档](./data/README.md)下载 [https://vulkan.gpuinfo.org/downloads/vulkan_asset_pack_gltf.zip](https://vulkan.gpuinfo.org/downloads/vulkan_asset_pack_gltf.zip) 并且解压到./data文件夹中

下面是相关的资料

- GLTF格式文档 https://github.com/KhronosGroup/glTF
- 带动画的GLTF模型已经上传到了目录data/buster_drone/busterDrone.gltf。这个gltf文件来自于 https://github.com/GPUOpen-LibrariesAndSDKs/Cauldron-Media/tree/v1.0.4/buster_drone
  - Buster Drone by LaVADraGoN, published under a Attribution-NonCommercial 4.0 International (CC BY-NC 4.0) license
  - 作者存放在sketchfab上展示的页面 https://sketchfab.com/3d-models/buster-drone-294e79652f494130ad2ab00a13fdbafd
- 完成这个作业需要额外学习的内容，都可以在作业框架下找到示例代码用于学习和参照（example code 是学习一个api最好的老师🙂）
  - 骨骼动画在这个工程下有可以学习的样例 examples/gltfskinning/gltfskinning.cpp
  - PBR材质 
    - 直接光照 examples/pbrbasic/pbrbasic.cpp 
    - 环境光照 examples/pbribl/pbribl.cpp

## 开做

作业过程中遇到对vulkan和框架不熟悉的地方，最好的方式就是看vulkan官方tutorial文档和examples示例代码！

[vulkan_pipeline](../vulkan_pipeline/vulkan_pipeline.md)

### 1.支持gltf的骨骼动画

![hw1_1](img/hw1_1.gif)
示例examples/gltfskinning/gltfskinning.cpp中的实现是有皮肤数据的，homework1中的gltf数据读出来是没有skin数据的。毕竟hw1中的模型是个机器飞机，没有皮肤才正常。

所以我们要做的就是修改Node结构体，增加Animation相关结构体，读取、存储、更新Animation，同时更新shader中的PushConsts model矩阵，vertex shader中通过mvp变换，即通过改变了顶点位置实现了骨骼动画，不用重新编译shader。

**关键点**：
在void updateAnimation(float deltaTime)中根据动画数据更新了每个相关Node的transform。注意示例中是通过updateJoints函数中来更新shader中的joint matrices，以实现骨骼动画。我们不需要updateJoints函数，通过在updateAnimation后修改model并pushConst给shader实现骨骼动画。
需要修改LoadNode函数，添加index的设置。

**方法一**：

1. 直接在drawNode中通过vkCmdPushConstants给GPU传递当前node的model值
2. 不需要修改默认的shader和descriptorset
3. 需要每一帧都调用buildCommandBuffers

```c++
// drawNode函数中，由于此代码在buildCommandBuffers()中调用，但是buildCommandBuffers并不会没帧调用，需要自行修改
glm::mat4 nodeMatrix = node->getLocalMatrix();
VulkanglTFModel::Node* currentParent = node->parent;
while (currentParent) {
    nodeMatrix = currentParent->getLocalMatrix() * nodeMatrix;
    currentParent = currentParent->parent;
}
// Pass the final matrix to the vertex shader using push constants
vkCmdPushConstants(commandBuffer, pipelineLayout, VK_SHADER_STAGE_VERTEX_BIT, 0, sizeof(glm::mat4), &nodeMatrix);
```

[方法一代码](hw1history/homework1_1.cpp)

**方法二**：

1. 顶点结构体添加NodeIndex，表示每个顶点属于哪个node
2. 使用着色器块存储缓冲区buffer存储和传递动画更新时每个node的变化矩阵
3. 在vertex shader中根据NodeIndex选择合适的model变化矩阵，变化顶点位置

### 2.支持gltf的PBR的材质(包括法线贴图)

1. 先学习了解gltf材质相关概念
   ![gltf](../vulkan_pipeline/img/gltfOverview-2.0.0b.png)
2. gltf的的每个材质，都有r、g、b和一组对应的纹理，扩充material结构体，保存材质属性和各个纹理的索引，并添加descriptorSet
3. 更新setupDescriptors()函数中的相关设置,images换成materials,添加额外的绑定
4. drawNode时，根据node-mesh-primitives下使用的材质索引，绑定对应材质的descriptorSet
5. 在pixel shader中使用材质属性和纹理更新颜色

### 3.Tone Mapping后处理(提高)

