---
layout: post
title: "Use Protobuf In Unity3D"
tags:
- unity3D
---

把.proto文件生成.cs文件需要下载工具:

```
sudo npm install -g protogen

brew install protobuf
```

测试文件 test.proto

```
message Person
{
    required int32 age = 1;
    required string name = 2;
}

message Family
{
    repeated Person person = 1;
}
```

命令行中生成.cs文件

```
protogen i:test.proto o:test.cs
```

下载protobuf-net工具:

https://github.com/mgravell/protobuf-net

将其中的protobuf-net.dll导入到Unity3D项目中的Plugins目录下
    
*听说这种方式在ios平台上会有问题，可以在网上找到相关处理*

将以上生成的test.cs文件导入Unity3D项目中

编写序列化、反序列化工具 `Serialization.cs`:

```
using ProtoBuf;
using System;
using System.IO;

public static class Serialization 
{
    public static byte[] Serialize<T>(T instance)
    {
        byte[] bytes;
        using (var ms = new MemoryStream())
        {
            Serializer.Serialize(ms, instance);
            bytes = new byte[ms.Position];
            var fullBytes = ms.GetBuffer();
            Array.Copy(fullBytes, bytes, bytes.Length);
        }
        return bytes;
    }

    public static T Deserialize<T>(object obj)
    {
        byte[] bytes = (byte[])obj;
        using (var ms = new MemoryStream(bytes))
        {
            return Serializer.Deserialize<T>(ms);
        }
    }
}
```

编写测试脚本 `Main.cs`:

```
using UnityEngine;
using System.Collections;
using test;

namespace ProtobufDemo
{
    public class Main : MonoBehaviour 
    {
        void Start () 
        {
            Person person = new Person();
            person.age = 28;
            person.name = "lcl";

            byte[] bytes = Serialization.Serialize<Person>(person);
            Debug.Log("序列化bytes length:" + bytes.Length);
            Person person2 = Serialization.Deserialize<Person>(bytes);
            Debug.Log("反列化 age:" + person2.age);
        }
    }
}
```