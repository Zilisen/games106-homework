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

need redo，将一些不太清晰的概念记录在下面。

## 在着色器中访问资源

### 描述符集

描述符集是作为整体绑定到管线的资源的集合。可以同时将多个集合绑定到一个管线。每一个集合都有一个布局，布局描述了集合中资源的排列顺序和类型。两个拥有相同布局的集合被视为兼容的和可相互交换的。描述符集的布局通过一个对象表示，集合都是参照这个对象创建的。另外，可被管线访问的集合的集合组成了另一个对象—— 管线布局。管线通过参照这个管线布局对象来创建。

![描述符集和管线集](img/2023-05-17-23-21-00.png)

在任何时刻，应用程序都可以将一个新的描述符集绑定到命令缓冲区，只要具有相同的布局就行。相同的描述符集布局可以用来创建多个管线。

可调用vkCreateDescriptorSetLayout()来创建描述符集布局对象，其原型如下：

```c++
VkResult vkCreateDescriptorSetLayout (
    VkDevice                                           device,
    constVkDescriptorSetLayoutCreateInfo＊      pCreateInfo,
    constVkAllocationCallbacks＊                  pAllocator,
    VkDescriptorSetLayout＊                          pSetLayout);
```

```c++
typedef structVkDescriptorSetLayoutCreateInfo {
    VkStructureType                     sType; // 设为 VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO
    const void＊                        pNext; // 设为 nullptr
    VkDescriptorSetLayoutCreateFlags    flags; // 留待以后使用，设为0
    uint32_t                            bindingCount; // 该集合中的绑定点个数
    constVkDescriptorSetLayoutBinding＊ pBindings; // 描述信息的数组的指针
} VkDescriptorSetLayoutCreateInfo;
```

```c++
// 资源绑定到描述符集里的绑定点
typedef structVkDescriptorSetLayoutBinding {
    uint32_t                 binding; // 绑定序号
    VkDescriptorType       descriptorType; // 资源类型
    uint32_t                 descriptorCount; // 
    VkShaderStageFlags     stageFlags;
    constVkSampler＊       pImmutableSamplers;
} VkDescriptorSetLayoutBinding;
```

为了把两个或多个描述符集打包成管线可以使用的形式，需要把它们汇集到一个VkPipelineLayout对象里。为此，需要调用vkCreatePipelineLayout():

```c++
VkResult vkCreatePipelineLayout (
    VkDevice                                     device,
    constVkPipelineLayoutCreateInfo＊      pCreateInfo, 
    constVkAllocationCallbacks＊            pAllocator,
    VkPipelineLayout＊                          pPipelineLayout);
```

```c++
// 管线布局创建结构体信息
typedef structVkPipelineLayoutCreateInfo {
    VkStructureType                     sType; // 设为VK_STRUCTURE_TYPE_PIPELINE_ LAYOUT_CREATE_INFO
    const void＊                          pNext; // 设为 nullptr
    VkPipelineLayoutCreateFlags      flags; // 设为 0
    uint32_t                             setLayoutCount; // 描述符集布局的数量（和管线布局中集合的个数相同
    constVkDescriptorSetLayout＊     pSetLayouts; // 一个指向VkDescriptorSetLayout类型句柄数组的指针（之前调用vkCreateDescriptorSetLayout()创建
    uint32_t                             pushConstantRangeCount; //推送常量
    constVkPushConstantRange＊       pPushConstantRanges;
} VkPipelineLayoutCreateInfo;
```

示例：

```glsl
#version450 core


layout(set = 0, binding= 0) uniform sampler2DmyTexture;
layout(set = 0, binding= 2) uniform sampler3DmyLut;
layout(set = 1, binding= 0) uniformmyTransforms
{
    mat4transform1;
    mat3transform2;
};


voidmain(void)
{
    // 什么都不做
}
```

```c++
//描述了合体的图像-采样器。其中有一个集合，两个互斥的绑定
static constVkDescriptorSetLayoutBinding Samplers[] =
{
    {
        0,                                              //相对于绑定的起始位置
        0
        VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, //合体的图像-采样器
        1,                                              //创建一个绑定
        VK_SHADER_STAGE_ALL,                          //在所有阶段里都能使用
        nullptr                                         //没有静态的采样器
    },
    {
        2,                                              //相对于绑定的起始位置,可以不连续
        2
        VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, //合体的图像-采样器
        1,                                              //创建一个绑定
        VK_SHADER_STAGE_ALL,                          //在所有阶段里都能使用
        nullptr                                         //没有静态的采样器
    }
};


//这是uniform块。其中有一个集合，一个绑定
static constVkDescriptorSetLayoutBinding UniformBlock =
{
    0,                                                   //相对于绑定的起始位置
    VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,               // Uniform内存块
    1,                                                   // 一个绑定
    VK_SHADER_STAGE_ALL,                               // 所有的阶段
    nullptr                                             // 没有静态的采样器
};
//现在创建两个描述符集布局
static constVkDescriptorSetLayoutCreateInfo createInfoSamplers =
{
    VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO,
    nullptr,
    0,
    2,
    &Samplers[0]
};
static constVkDescriptorSetLayoutCreateInfo createInfoUniforms =
{
    VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO,
    nullptr,
    0,
    1,
    &UniformBlock
};
//该数组持有两个集合布局
VkDescriptorSetLayout setLayouts[2];


vkCreateDescriptorSetLayout(device, &createInfoSamplers,
                              nullptr, &setLaouts[0]);
vkCreateDescriptorSetLayout(device, &createInfoUniforms,
                              nullptr, &setLayouts[1]);
//现在创建管线布局
constVkPipelineLayoutCreateInfo pipelineLayoutCreateInfo =
{
    VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO, nullptr,
    0,
    2, setLayouts,
    0, nullptr
};


VkPipelineLayout pipelineLayout;


vkCreatePipelineLayout(device, &pipelineLayoutCreateInfo,
                        nullptr, pipelineLayout);
```

当管线布局不再需要时，应该调用vkDestroyPipelineLayout()来销毁它。这将释放与管线布局对象关联的任何资源。

```c++
voidvkDestroyPipelineLayout (
    VkDevice                               device,
    VkPipelineLayout                     pipelineLayout,
    constVkAllocationCallbacks＊      pAllocator);
```

在销毁管线布局对象后，不应再次使用它了。然而，通过该管线布局对象创建的管线依然是有效的，直至它们被销毁。因此，没有必要在创建了管线之后依然让管线布局对象继续存在。

描述符集布局同上。描述符集布局被销毁后，它的句柄就失效了，就不应该再次使用了。然而，引用这个集合创建的描述符集、管线布局和其他的对象仍旧有效。

```c++
voidvkDestroyDescriptorSetLayout (
    VkDevice                             device,
    VkDescriptorSetLayout             descriptorSetLayout,
    constVkAllocationCallbacks＊     pAllocator);
```

### 绑定资源到描述符集

资源是通过描述符表示的，绑定到管线上的顺序是：首先把描述符绑定到集合上，然后把描述符集绑定到管线。这样花费很少时间就能绑定一个很大的资源，因为被特定绘制命令使用的资源的集合可以提前确定，并可以预先创建存放它们的描述符集。

描述符是从“描述符池”分配出来的。因为不管在哪个实现上对不同资源类型的描述符很可能都拥有相同的数据结构，所以池化分配可以让驱动更加高效地使用内存。可以使用vkCreateDescriptorPool()来创建描述符缓存池：

```c++
VkResult vkCreateDescriptorPool (
    VkDevice                                     device,
    constVkDescriptorPoolCreateInfo＊      pCreateInfo,
    constVkAllocationCallbacks＊            pAllocator,
    VkDescriptorPool＊                          pDescriptorPool);
```

```c++
typedef structVkDescriptorPoolCreateInfo {
    VkStructureType                    sType; // VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO
    const void＊                        pNext; // nullptr
    VkDescriptorPoolCreateFlags     flags; // 0
    uint32_t                            maxSets; // 指定了可从池中分配的集合数量的最大值
    uint32_t                            poolSizeCount; // 数组大小
    constVkDescriptorPoolSize＊     pPoolSizes; // 指定了可以存储在集合中的每种类型资源可用的描述符个数 数组
} VkDescriptorPoolCreateInfo;
```

```c++
typedef structVkDescriptorPoolSize {
    VkDescriptorType     type; // 指定了资源的类型
    uint32_t               descriptorCount; // 指定了在池中该种类型资源的个数
} VkDescriptorPoolSize;
```

如果pPoolSizes数组中没有指定某个特殊类型的资源的元素，那么就不能从池里分配那种类型的描述符。如果一个特定资源类型在数组中出现了两次，那么使用它们两个的字段descriptorCount之和为该类型指定缓存池的大小。池里的资源总数在从该池里分配的集合之间划分。

为了从缓存池中分配多个描述符块，可以调用vkAllocateDescriptorSets()来创建描述符集对象：

```c++
VkResult vkAllocateDescriptorSets (
    VkDevice                                      device,
    constVkDescriptorSetAllocateInfo＊      pAllocateInfo,
    VkDescriptorSet＊                            pDescriptorSets);
```

```c++
typedef structVkDescriptorSetAllocateInfo {
    VkStructureType                     sType; // VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO
    const void＊                          pNext; // nullptr
    VkDescriptorPool                    descriptorPool; // 指定可以从中分配集合的描述符缓存池的句柄
    uint32_t                            descriptorSetCount; // 创建的集合个数
    constVkDescriptorSetLayout＊     pSetLayouts; // 每个集合的布局，数组
} VkDescriptorSetAllocateInfo;
```

如果在创建描述符缓存池时结构体VkDescriptorSetCreateInfo的成员flags包含VK_DESCRIPTOR_POOL_CREATE_FREE_DESCRIPTOR_SET_BIT标志位，那么描述符集可以通过释放返回缓存池。可调用vkFreeDescriptorSets()来释放一个或者多个描述符集：

```c++
VkResult vkFreeDescriptorSets (
    VkDevice                                  device,
    VkDescriptorPool                        descriptorPool,
    uint32_t                                  descriptorSetCount,
    constVkDescriptorSet＊                 pDescriptorSets);
```

即使在创建描述符缓存池时VK_DESCRIPTOR_POOL_CREATE_FREE_DESCRIPTOR_SET_BIT没有指定，也可以从缓存池分配的所有集合回收所有的资源。可以调用vkResetDescriptorPool()函数并重置缓存池来实现该操作。有了这条命令，就没有必要显式地指定从缓存池中分配的每一个集合:

```c++
VkResult vkResetDescriptorPool (
    VkDevice                                  device,
    VkDescriptorPool                        descriptorPool,
    VkDescriptorPoolResetFlags            flags // 0
    );
```

为了彻底释放和资源关联的描述符缓存池，应该通过调用vkDestroyDescriptorPool()来销毁缓冲池对象:

```c++
void vkDestroyDescriptorPool(
    VkDevice                                  device,
    VkDescriptorPool                        descriptorPool,
    constVkAllocationCallbacks＊          pAllocator);
```

当销毁了描述符缓存池时，它里面所有的资源也释放了，包含从它分配的任何集合。

可以直接写入描述符集或者从另一个描述符集复制绑定，来把资源绑定到描述符集。两种情况下，都使用vkUpdateDescriptorSets()命令:

```c++
voidvkUpdateDescriptorSets (
    VkDevice                                 device,
    uint32_t                                 descriptorWriteCount, // 直接写入的个数
    constVkWriteDescriptorSet＊          pDescriptorWrites, // 写入的数据
    uint32_t                                 descriptorCopyCount, // 描述符复制的次数
    constVkCopyDescriptorSet＊           pDescriptorCopies // 复制的数据
    );
```

```c++
typedef structVkWriteDescriptorSet {
    VkStructureType                        sType; // VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET
    const void＊                             pNext; // nullptr
    VkDescriptorSet                        dstSet; // 目标描述符集
    uint32_t                                 dstBinding; // 绑定索引
    uint32_t                                 dstArrayElement; // 如果集合中的绑定引用了一个资源类型的数组，那么dstArrayElement用来指定更新起始的索引,否则为0
    uint32_t                                 descriptorCount; // 指定需要更新的连续描述符个数,非数组为1
    VkDescriptorType                       descriptorType; // 更新的资源的类型，决定了下一个参数
    constVkDescriptorImageInfo＊         pImageInfo;
    constVkDescriptorBufferInfo＊       pBufferInfo;
    constVkBufferView＊                    pTexelBufferView;
} VkWriteDescriptorSet;
```

```c++
typedef structVkDescriptorImageInfo {
    VkSampler          sampler;
    VkImageView       imageView;
    VkImageLayout     imageLayout;
} VkDescriptorImageInfo;
```

将要绑定到描述符集的图像视图的句柄通过imageView传递。如果描述符集的资源是VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER，那么伴随的采样器的句柄通过sampler指定。当在描述符集里使用图像时，所期望的布局通过imageLayout传递。

```c++
typedef structVkDescriptorBufferInfo {
    VkBuffer          buffer;
    VkDeviceSize     offset;
    VkDeviceSize     range;
} VkDescriptorBufferInfo;
```

将要绑定的缓冲区对象通过buffer指定，绑定的起始位置与大小（都以字节为单位）分别通过offset和range指定。绑定范围必须在缓冲区对象之内。为了绑定整个缓冲区（从缓冲区对象推断范围的大小），需要把range设置为VK_WHOLE_SIZE。

除了向描述符集直接写入之外，vkUpdateDescriptorSets()可以从一个描述符集向另外一个集合里复制描述符，或者在一个集合中的不同绑定之间复制描述符:

```c++
typedef structVkCopyDescriptorSet {
    VkStructureType     sType; // VK_STRUCTURE_TYPE_COPY_DESCRIPTOR_SET
    const void＊          pNext; // nullptr
    VkDescriptorSet     srcSet; // 源描述符集
    uint32_t             srcBinding;
    uint32_t             srcArrayElement;
    VkDescriptorSet     dstSet; // 目标描述符集
    uint32_t             dstBinding;
    uint32_t             dstArrayElement;
    uint32_t             descriptorCount;
} VkCopyDescriptorSet;
```

这两个可以是同一个集合，只要将要复制的描述符的区域不重叠即可。字段srcBinding和dstBinding分别指定了源与目标描述符的绑定索引。如果将要复制的描述符形成了一个绑定数组，源和目标描述符区域的起始位置的索引分别通过srcArrayElement与dstArrayElement指定；如果描述符并不形成数组，这两个字段可以设置为0。需要复制的描述符数组的长度通过descriptorCount指定。如果复制的数据不在描述符数组中，那么descriptorCount应设置为1。

### 绑定描述符集

和管线一样，要访问描述符集附带的资源，描述符集必须要绑定到命令缓冲区，而该缓冲区将执行访问这些描述符的命令。描述符集也有两个绑定点—— 一个是计算，一个是图形——这些集合将会被对应类型的管线访问到。

可调用vkCmdBindDescriptorSets()来把描述符集绑定到一个命令缓冲区上：

```c++
void vkCmdBindDescriptorSets (
    VkCommandBuffer               commandBuffer, // 描述符集将要绑定的命令缓冲区
    VkPipelineBindPoint          pipelineBindPoint, // VK_PIPELINE_BIND_POINT_COMPUTE或VK_PIPELINE_BIND_POINT_GRAPHICS
    VkPipelineLayout             layout, // 指定管线（将访问描述符）将要使用的管线布局
    uint32_t                       firstSet,
    uint32_t                       descriptorSetCount,
    constVkDescriptorSet＊      pDescriptorSets,
    uint32_t                       dynamicOffsetCount, // 将要设置的动态偏移量的个数
    constuint32_t＊               pDynamicOffsets
    );
```

要绑定管线布局可访问的集合的一个子集，可使用参数firstSet与descriptorSetCount分别指定第一个集合的索引和集合的个数。pDescriptorSets是一个指针，指向VkDescriptorSet类型句柄的数组，这些句柄指向将要绑定的集合。可通过之前讲过的vkAllocateDescriptorSets()调用来获取这些句柄。

### uniform、纹素和存储缓冲区

着色器可以直接通过3种资源访问缓冲区内存的内容。

- uniform块提供了对存储在缓冲区对象中常量（只读）数据的快速访问。在着色器内就像结构体一样声明它们，使用绑定到描述符集的缓冲区资源来将这些uniform块绑定到内存上。
- 着色器块提供了对缓冲区对象的读写访问。和uniform块的声明类似，数据就像结构体一样组织，但可以写入数据。着色器块也支持原子操作。
- 纹素缓冲区提供了对存储格式化纹素数据的长线性数组的访问能力。它们是只读的，纹素缓冲区绑定会将潜在的数据格式转化为着色器读取缓冲区时期待的浮点形式。

```glsl
//uniform块
layout(set = 0, binding= 1) uniform my_uniform_buffer_t
{
    float foo;
    vec4 bar;
    int baz[42];
} my_uniform_buffer;

//着色器块
layout(set = 0, binding= 2) buffer my_storage_buffer_t
{
    int peas;
    float carrots;
    vec3 potatoes[99];
} my_storage_buffer;

//纹素缓冲区
layout(set = 0, binding= 3) uniform samplerBuffer my_float_texel_buffer;
layout(set = 0, binding= 4) uniform isamplerBuffer my_signed_texel_buffer;
layout(set = 0, binding= 5) uniform usamplerBuffer my_unsigned_texel_buffer;
```

### 推送常量

推送常量是一种着色器里使用的uniform变量，可以像uniform块那样使用，但是并不需要存储在内存里，它由Vulkan自身持有和更新。结果就是，这些常量的新值可以被直接从命令缓冲区推送到管线，因此而得名。

```c++
typedef struct VkPushConstantRange {
    VkShaderStageFlags     stageFlags;
    uint32_t                 offset;
    uint32_t                 size;
} VkPushConstantRange;
```

要在管线内使用推送常量，就需要在管线着色器内声明变量来表示它们。在SPIR-V着色器中，推送常量通过在变量声明上使用PushConstant存储类来声明。在GLSL中，这样的声明可以通过声明一个带有push_constant限定符的uniform块来产生。在每一个管线中只有一个这样的块声明。逻辑上，它和std430块有相同的“内存内”布局。然而，这个布局只用来计算成员的偏移量，并且可能并不是Vulkan实现在内部表示块内数据的方式。

```glsl
layout(push_constant) uniform my_push_constants_t
{
    int bourbon;
    int scotch;
    int beer;
} my_push_constants;
```

推送常量变成了使用它的管线布局的一部分。当把推送常量包含进管线时，它们也许会消耗Vulkan用于追踪管线或描述符绑定的一些资源。因此，应该把推送常量看作相当珍贵的资源。

要更新一个或者多个推送常量，可调用vkCmdPushConstants():

```c++
void vkCmdPushConstants (
    VkCommandBuffer                 commandBuffer,
    VkPipelineLayout                layout,
    VkShaderStageFlags             stageFlags,
    uint32_t                          offset,
    uint32_t                          size,
    const void＊                      pValues);
```

将执行更新的命令缓冲区通过commandBuffer指定，定义推送常量的位置的布局通过layout指定。这个布局必须要和随后绑定的并用于分发或者绘制命令的任何管线相兼容。
推送常量在逻辑上按照std430布局规则在内存中存储，每一个推送常量的内容都“生存”在相对于内存块的起始位置的偏移量上（可以使用std430规则计算）。要更新的第一个常量在虚拟块内的偏移量由offset指定，更新的量的大小通过size指定，单位是字节。
需要放到推送常量中的数据通过一个指针pValues传递。
把推送常量当作稀缺资源。优先使用正常的uniform块来存储大数据结构，将推送常量用作整数或者更新非常频繁的数据。
