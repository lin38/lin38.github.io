---
title: 序列化-2-protobuf实现方式
date: 2018-08-10 09:31:35
categories: 序列化
tags: 
  - protobuf
toc: true
list_number: false
---



# 1、下载并配置环境变量

​	下载地址如下：https://github.com/google/protobuf



# 2、编写proto文件

```protobuf
syntax = "proto3";
option java_package = "io.github.lin38.serialize.demo006.model";
option java_outer_classname = "PlayerModule";

message PBPlayer{
	int64 playerId = 1;
	
	int32 age = 2;
	
	string name = 3;
	
	repeated int32 skills = 4;
}

message PBResource{
	int64 gold = 1;
	
	int32 energy = 2;
}
```

<!--more-->



# 3、执行protoc命令生成java文件

```shell
protoc -I=proto --java_out=../../../../../ proto/*.proto
```

命令说明：

-I 后面是proto文件所在的目录，–java_out 后面是生成java文件存放地址  最后一行是proto文件的名称，可以写绝对地址，也可以直接写proto文件名称



# 4、添加maven依赖

```xml
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>3.5.1</version>
</dependency>
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java-util</artifactId>
    <version>3.5.1</version>
</dependency>
```



# 5、测试

```java
package io.github.lin38.serialize.demo006;

import java.util.Arrays;

import com.google.protobuf.InvalidProtocolBufferException;
import com.google.protobuf.util.JsonFormat;

import io.github.lin38.serialize.demo006.model.PlayerModule;
import io.github.lin38.serialize.demo006.model.PlayerModule.PBPlayer;
import io.github.lin38.serialize.demo006.model.PlayerModule.PBPlayer.Builder;

public class Test {
	public static void main(String[] args) throws InvalidProtocolBufferException {
		
		// 序列化
		Builder builder = PlayerModule.PBPlayer.newBuilder();
		builder.setPlayerId(1000);
		builder.setAge(20);
		builder.setName("peter");
		builder.addSkills(101);
		
		PBPlayer player = builder.build();
		byte[] bs = player.toByteArray();
		System.out.println("序列化完成，结果如下：");
		System.out.println(Arrays.toString(bs));
		
		// 反序列化
		System.out.println("反序列化完成，结果如下：");
		PBPlayer player2 = PlayerModule.PBPlayer.parseFrom(bs);
//		System.out.println(player2);
		System.out.println(JsonFormat.printer().print(player2));
		
	}
}
```

执行结果：

```
序列化完成，结果如下：
[8, -24, 7, 16, 20, 26, 5, 112, 101, 116, 101, 114, 34, 1, 101]
反序列化完成，结果如下：
{
  "playerId": "1000",
  "age": 20,
  "name": "peter",
  "skills": [101]
}
```

