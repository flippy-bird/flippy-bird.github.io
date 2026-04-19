---
title: 开发一个Blender插件(1)
date: 2023-03-05 14:44:06
description: "blender插件开发"
tags:
    - Blender
categories:
    - Blender插件开发
---



### Blender 模块

要是用blender模块，首先要导入blender库

```Python
import bpy
```

![image-20260305145458917](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260305145458917.png)

#### Data

#### Context

bpy.context

- 只能访问 当前激活 的 area 中的数据
- 获取到的数据是readonly（只读的）

跨域的访问

context.scene

#### Operators

bpy.ops

所有的操作指令，像按钮，建模操作，删除创建模型。。。

#### Types

bpy.types

所有实现的“class类模板”定义

#### Props

bpy.props

blender中的properties（属性）管理

##### RNA属性

- 对bpy.types模板类的属性进行扩充
- 所有的bpy.types模板的子类均有该属性
- 需要提前“声明”
- Int, float, string, bool, enum

```Python
bpy.props.IntProperty
bpy.props.FloatProperty
bpy.props.EnumProperty
# 
bpy.props.PointerProperty

# 数组
bpy.props.IntVectorProperty
bpy.props.FloatVectorProperty

bpy.props.removeProperty
import bpy

bpy.types.Object.myInt = bpy.props.IntProperty(
           name = 'test',
           max = 10, 
           min = 1, 
           default = 5
 )
 
 # 对所有object都有该属性
 
 bpy.types.Mesh.myFloat = bpy.props.FloatProperty(
           name = 'float',
           max = 10.0,
           min = -10.0,
           default = 1
 )
 
 #  对所有Mesh均有该属性
 
```

##### ID属性

- 对bpy.types 模板实例类的属性进行扩展
- 只有单一的实例类才能拥有该属性
- 不要要提前“声明”
- Int, float, string. (没有enum和bool)

```Python
import bpy

bpy.data.objects['Cube']['idString'] = 'hello cube'
# 只针对‘Cube’这个物体有idString属性
```

#### Utils

bpy.utils

Blender 自身的工具

一般用到的是

```Python
bpy.utils.register_class()
bpy.utils.unregirster_class()
```

#### Path

bpy.path

系统文件系统的访问

bpy.path.abspath

bpy.path.basename

bpy.path.relpath

### Blender UI开发

#### 命名规范

Operator 命名规范 ： __OT__

Menu 命名规范：__MT__

header命名规范：__HT__

panel命名规范：__PT__

### 其它

```Python

```
