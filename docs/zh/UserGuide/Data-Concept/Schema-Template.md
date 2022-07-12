<!--

    Licensed to the Apache Software Foundation (ASF) under one
    or more contributor license agreements.  See the NOTICE file
    distributed with this work for additional information
    regarding copyright ownership.  The ASF licenses this file
    to you under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License.  You may obtain a copy of the License at
    
        http://www.apache.org/licenses/LICENSE-2.0
    
    Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied.  See the License for the
    specific language governing permissions and limitations
    under the License.

-->

# 元数据模板

## 问题背景

对于大量的同类型的实体，每一个实体下的物理量都相同，为每个序列注册时间序列一方面时间序列的元数据将占用较多的内存资源，另一方面，大量序列的维护工作也会十分复杂。

为了实现同类型不同实体的物理量元数据共享，减少元数据内存占用，同时简化同类型实体的管理，IoTDB引入元数据模板功能。

下图展示了一个燃油车场景的数据模型，各地区的多台燃油车的速度、油量、加速度、角速度四个物理量将会被采集，显然这些燃油车实体具备相同的物理量。

<img style="width:100%; max-width:800px; max-height:600px; margin-left:auto; margin-right:auto; display:block;" src="https://github.com/apache/iotdb-bin-resources/blob/main/docs/UserGuide/Data%20Concept/Measurement%20Template/example_without_template.png?raw=true" alt="example without template">

## 概念定义

元数据模板（Schema template）

实际应用中有许多实体所采集的物理量相同，即具有相同的工况名称和类型，可以声明一个**元数据模板**来定义可采集的物理量集合。

将元数据模版挂载在树形数据模式的任意节点上，表示该节点下的所有实体具有相同的物理量集合。

目前每一条路径节点仅允许挂载一个元数据模板，即当一个节点被挂载元数据模板后，它的祖先节点和后代节点都不能再挂载元数据模板。实体将使用其自身或祖先的元数据模板作为有效模板。

特别需要说明的是，挂载模板与使用模板的概念不同。一个节点挂载模板后，其所有后代节点都可以使用这个模板，因此可以通过向同类实体的祖先节点挂载模板来简化操作。当系统向挂载模板的节点（或其后代节点）插入模板中定义的物理量时，这个节点就被设置为“正在使用模板”。

使用元数据模板后，问题背景中示例的燃油车数据模型将会转变至下图所示的形式。所有的物理量元数据仅在模板中保存一份，所有的实体共享模板中的元数据。

<img style="width:100%; max-width:800px; max-height:600px; margin-left:auto; margin-right:auto; display:block;" src="https://github.com/apache/iotdb-bin-resources/blob/main/docs/UserGuide/Data%20Concept/Measurement%20Template/example_with_template.png?raw=true" alt="example with template">

## 使用

目前，用户可以通过 Session 编程接口或 IoTDB-SQL 来使用元数据模板，包括模板的创建、修改、挂载与卸载等。Session 编程接口的详细文档可参见[此处](../API/Programming-Java-Native-API.md)，IoTDB-SQL 的详细文档可参加[此处](../Operate-Metadata/Template.md)。下文将以 Session 中使用方法为例，进行简要介绍。


* 创建元数据模板

在 Session 中创建元数据模板，可以通过先后创建 Template、MeasurementNode 的对象，构造模板内部物理量结构，并通过以下接口创建模板

```java
public void createSchemaTemplate(Template template);

Class Template {
    private String name;
    private boolean directShareTime;
    Map<String, Node> children;
    public Template(String name, boolean isShareTime);
    
    public void addToTemplate(Node node);
    public void deleteFromTemplate(String name);
    public void setShareTime(boolean shareTime);
}

Abstract Class Node {
    private String name;
    public void addChild(Node node);
    public void deleteChild(Node node);
}

Class MeasurementNode extends Node {
    TSDataType dataType;
    TSEncoding encoding;
    CompressionType compressor;
    public MeasurementNode(String name, 
                           TSDataType dataType, 
                           TSEncoding encoding,
                          CompressionType compressor);
}
```

* 构造与挂载元数据模板

构造上图中的元数据模板，并挂载到对应节点，可参考如下代码。**请注意，我们强烈建议您将模板设置在存储组或存储组下层的节点中，以更好地适配未来地更新及各模块的协作。**

``` java
MeasurementNode nodeV = new MeasurementNode("velocity", TSDataType.FLOAT, TSEncoding.RLE, CompressionType.SNAPPY);
MeasurementNode nodeF = new MeasurementNode("fuel_amount", TSDataType.FLOAT, TSEncoding.RLE, CompressionType.SNAPPY);
MeasurementNode nodeA = new MeasurementNode("acceleration", TSDataType.DOUBLE, TSEncoding.GORILLA, CompressionType.SNAPPY);
MeasurementNode nodeAng = new MeasurementNode("angular_velocity", TSDataType.DOUBLE, TSEncoding.GORILLA, CompressionType.SNAPPY);

Template template = new Template("template");
template.addToTemplate(nodeV);
template.addToTemplate(nodeF);
template.addToTemplate(nodeA);
template.addToTemplate(nodeAng);

createSchemaTemplate(template);
setSchemaTemplate("template", "root.Beijing");
```

**挂载元数据模板后，即可向挂载节点或该节点的子孙节点，按照模板的模式进行数据写入**。例如，按上述代码创建并挂载模板，并在 root.Beijing 路径上设置了存储组后，即可写入例如 root.Beijing.petro_vehicle.velocity 等时间序列数据，系统将自动创建 petro_vehicle 节点，并设置其“正在使用模板”，对写入数据应用模板中为 velocity 定义的元数据信息。

* 修改、激活、解除、卸载与删除元数据模板

在元数据模板相关的操作中，除了上文中提到的创建、挂载，还有一些高级功能可以使用。具体细节可以参考[编程接口说明](../API/Programming-Java-Native-API.md)和[SQL命令说明](../Operate-Metadata/Template.md)，在此仅对其主要概念进行简要介绍。

创建元数据模板后，还可以增加或删除模板中的节点。需要注意的是，一旦模板被挂载，在卸载模板之前就不再允许从模板中删除节点。

对于挂载了模板的节点或其子孙节点，可以通过相应接口激活模板。**对挂载模板与激活模板的状态进行区分，是为了服务一种常见的场景**：在 Apache IoTDB 元数据模型 MTree 中，经常需要在数量众多的节点上“应用”元数据模板，而这些节点一般拥有共同的祖先节点。因此，可以在其共同祖先节点**挂载**模板，而不必对其大量的孩子节点进行挂载操作。对于需要“应用”模板的节点，则应该使用**激活模板**的操作。

解除与卸载模板，分别是激活与挂载模板的逆过程。**其中，对一个节点解除模板还会删除该节点在模板中的序列写入的数据。**需要注意的是，如果一个挂载了模板的节点或其子孙节点，当前已经激活了模板，那么在解除激活之前，该节点就不能卸载模板。类似地，如果有任意一个节点挂载了模板，那么在其卸载之前，模板就不能被删除。

上述功能均有对应权限进行控制，具体可以参见[权限管理](../Administration-Management/Administration.md)。