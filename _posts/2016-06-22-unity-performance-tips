---
layout: post
title: "Unity Performance Tips"
tags:
- unity
---

Unity资源管理机制

避免内存泄露
减少内存分配

反射


unload(false) 意味着你可以再次加载同一个资源了，
这可能导致同一份资源加载多个到内存

使用压缩纹理

对于偏数据的数据结构，可以考虑使用struct替代class,
这样很多时候就避免了gcalloc

一些会产生GC的操作:
	生成一个新的委托，例如将方法做为参数传入

	对List进行foreach

	用枚举做Key进行字典查找

	访问animation等组件

	获取SkinedMeshRenderer.bones或Mesh.uvs之类的属性

	yield return 0 (建议全部替换为yield return null)

	调用GetComponentInChildren

	不要直接访问gameObject的tag属性
	go.tag == "human" => go.CompareTag("human")

string连接
建议使用StringBuilder 或者 String.Format

uGUI的Image.Set_FillAmount有gc alloc消耗，建议
赋值之前做一次是否改变的判断

UnityEvent.Invoke性能很差, 建议使用Delegate

优化性能时一定要杜绝每帧gc alloc的实现

transform 引用保留

## 装箱之所以会带来性能损耗，因为它需要完成下面三个步骤
1. 首先，会为值类型在托管堆中分配内存。除了值类型本身所分配的内存外，内存总量还要加上
	类型指针和同步块索引所占用的内存
2. 将值类型的值复制到新分配的堆内存中。
3. 返回已经成为引用类型的对象的地址