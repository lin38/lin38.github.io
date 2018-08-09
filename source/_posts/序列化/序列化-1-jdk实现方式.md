---
title: 序列化-1-jdk实现方式
date: 2018-08-09 15:43:49
categories: 序列化
tags: 
  - transient
  - Serializable
  - Externalizable
  - serialVersionUID
toc: true
list_number: false
---

# 1、序列化与反序列化

　　把对象转换为字节序列的过程称为对象的序列化。
　　把字节序列恢复为对象的过程称为对象的反序列化。
　　对象的序列化主要有两种用途：
　　1） 把对象的字节序列永久地保存到硬盘上，通常存放在一个文件中；
　　2） 在网络上传送对象的字节序列。

　　在很多应用中，需要对某些对象进行序列化，让它们离开内存空间，入住物理硬盘，以便长期保存。比如最常见的是Web服务器中的Session对象，当有 10万用户并发访问，就有可能出现10万个Session对象，内存可能吃不消，于是Web容器就会把一些session先序列化到硬盘中，等要用了，再把保存在硬盘中的对象还原到内存中。

　　当两个进程在进行远程通信时，彼此可以发送各种类型的数据。无论是何种类型的数据，都会以二进制序列的形式在网络上传送。发送方需要把这个Java对象转换为字节序列，才能在网络上传送；接收方则需要把字节序列再恢复为Java对象。



# 2、JDK关于序列化的API

　　java.io.ObjectOutputStream代表对象输出流，它的writeObject(Object obj)方法可对参数指定的obj对象进行序列化，把得到的字节序列写到一个目标输出流中。

​	java.io.ObjectInputStream代表对象输入流，它的readObject()方法从一个源输入流中读取字节序列，再把它们反序列化为一个对象，并将其返回。

​	只有实现了Serializable和Externalizable接口的类的对象才能被序列化。Externalizable接口继承自 Serializable接口，实现Externalizable接口的类完全由自身来控制序列化的行为，而仅实现Serializable接口的类可以 采用默认的序列化方式 。

​	对象序列化包括如下步骤：

​	1） 创建一个对象输出流，它可以包装一个其他类型的目标输出流，如文件输出流；

​	2） 通过对象输出流的writeObject()方法写对象。

​	对象反序列化的步骤如下：

​	1） 创建一个对象输入流，它可以包装一个其他类型的源输入流，如文件输入流；

​	2） 通过对象输入流的readObject()方法读取对象。

<!--more-->



# 3、Serializable

<center>示例1-序列化java对象到数组中，并从数组反序列化对象</center>

```java
package io.github.lin38.serialize.demo001;

import java.io.Serializable;
import java.util.ArrayList;
import java.util.List;

public class Player implements Serializable {

	private static final long serialVersionUID = -5248069984631225347L;

	private long playerId;

	private int age;

	private String name;

	private List<Integer> skills = new ArrayList<>();

	public Player(long playerId, int age, String name) {
		this.playerId = playerId;
		this.age = age;
		this.name = name;
	}

	// 省略getter、setter
}
```

```java
package io.github.lin38.serialize.demo001;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.util.Arrays;

/**
 * jdk方式实现序列化与反序列化-写入数组
 */
public class JAVA2Bytes {

	public static void main(String[] args) throws Exception {
		byte[] bytes = toBytes();
		toPlayer(bytes);
	}

	/**
	 * 序列化
	 */
	public static byte[] toBytes() throws IOException {

		Player player = new Player(101, 20, "peter");
		player.getSkills().add(1001);

		ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
		ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);

		// 写入对象
		objectOutputStream.writeObject(player);

		// 获取字节数组
		byte[] byteArray = byteArrayOutputStream.toByteArray();
		System.out.println("序列化之后的内容...");
		System.out.println(Arrays.toString(byteArray));
		return byteArray;
	}

	/**
	 * 反序列化
	 */
	public static void toPlayer(byte[] bs) throws Exception {

		ObjectInputStream inputStream = new ObjectInputStream(new ByteArrayInputStream(bs));
		Player player = (Player) inputStream.readObject();

		// 打印
		System.out.println("反序列化之后的内容...");
		System.out.println("playerId:" + player.getPlayerId());
		System.out.println("age:" + player.getAge());
		System.out.println("name:" + player.getName());
		System.out.println("skills:" + (Arrays.toString(player.getSkills().toArray())));
	}

}
```

执行结果：

```
序列化之后的内容...
[-84, -19, 0, 5, 115, 114, 0, 40, 105, 111, 46, 103, 105, 116, 104, 117, 98, 46, 108, 105, 110, 51, 56, 46, 115, 101, 114, 105, 97, 108, 105, 122, 101, 46, 100, 101, 109, 111, 48, 48, 49, 46, 80, 108, 97, 121, 101, 114, -73, 43, 28, 39, -119, -86, -125, -3, 2, 0, 4, 73, 0, 3, 97, 103, 101, 74, 0, 8, 112, 108, 97, 121, 101, 114, 73, 100, 76, 0, 4, 110, 97, 109, 101, 116, 0, 18, 76, 106, 97, 118, 97, 47, 108, 97, 110, 103, 47, 83, 116, 114, 105, 110, 103, 59, 76, 0, 6, 115, 107, 105, 108, 108, 115, 116, 0, 16, 76, 106, 97, 118, 97, 47, 117, 116, 105, 108, 47, 76, 105, 115, 116, 59, 120, 112, 0, 0, 0, 20, 0, 0, 0, 0, 0, 0, 0, 101, 116, 0, 5, 112, 101, 116, 101, 114, 115, 114, 0, 19, 106, 97, 118, 97, 46, 117, 116, 105, 108, 46, 65, 114, 114, 97, 121, 76, 105, 115, 116, 120, -127, -46, 29, -103, -57, 97, -99, 3, 0, 1, 73, 0, 4, 115, 105, 122, 101, 120, 112, 0, 0, 0, 1, 119, 4, 0, 0, 0, 1, 115, 114, 0, 17, 106, 97, 118, 97, 46, 108, 97, 110, 103, 46, 73, 110, 116, 101, 103, 101, 114, 18, -30, -96, -92, -9, -127, -121, 56, 2, 0, 1, 73, 0, 5, 118, 97, 108, 117, 101, 120, 114, 0, 16, 106, 97, 118, 97, 46, 108, 97, 110, 103, 46, 78, 117, 109, 98, 101, 114, -122, -84, -107, 29, 11, -108, -32, -117, 2, 0, 0, 120, 112, 0, 0, 3, -23, 120]
反序列化之后的内容...
playerId:101
age:20
name:peter
skills:[1001]
```



<center>示例1-序列化java对象到文件中，并从文件反序列化对象</center>

```java
package io.github.lin38.serialize.demo002;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.util.Arrays;

/**
 * jdk方式实现序列化与反序列化-写入文件
 */
public class JAVA2File {

	public static void main(String[] args) throws Exception {
		serialize();
		deserialize();
	}

	/**
	 * 序列化
	 */
	public static void serialize() throws IOException {
		Player player = new Player(101, 20, "peter");
		player.getSkills().add(1001);

		FileOutputStream fileOutputStream = new FileOutputStream("F:/palyer.data");
		ObjectOutputStream objectOutputStream = new ObjectOutputStream(fileOutputStream);

		// 写入对象
		objectOutputStream.writeObject(player);
		objectOutputStream.close();
	}

	/**
	 * 反序列化
	 */
	public static void deserialize() throws Exception {
		FileInputStream fileInputStream = new FileInputStream("F:/palyer.data");
		ObjectInputStream inputStream = new ObjectInputStream(fileInputStream);
		Player player = (Player) inputStream.readObject();

		// 打印
		System.out.println("playerId:" + player.getPlayerId());
		System.out.println("age:" + player.getAge());
		System.out.println("name:" + player.getName());
		System.out.println("skills:" + (Arrays.toString(player.getSkills().toArray())));
		
		inputStream.close();
	}

}
```



# 4、transient关键字

​	使用Serializable方式实现序列化时，如果序列化对象中某些字段被transient关键字修饰，则表示这些字段不会被序列化。所以在反序列化时候，也不会获取到这些字段的值。

<center>示例：</center>

```java
package io.github.lin38.serialize.demo003;

import java.io.Serializable;
import java.util.ArrayList;
import java.util.List;

public class Player implements Serializable {

	private static final long serialVersionUID = -5248069984631225347L;

	private long playerId;

	private int age;

	private String name;
	
	private transient String gender;

	private List<Integer> skills = new ArrayList<>();

	public Player(long playerId, int age, String name, String gender) {
		this.playerId = playerId;
		this.age = age;
		this.name = name;
		this.gender = gender;
	}

	// 省略getter、setter
}
```

```java
package io.github.lin38.serialize.demo003;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.util.Arrays;

/**
 * jdk方式实现序列化与反序列化-transient关键字修饰的字段不参与序列化
 */
public class JAVA2Bytes {

	public static void main(String[] args) throws Exception {
		byte[] bytes = toBytes();
		toPlayer(bytes);
	}

	/**
	 * 序列化
	 */
	public static byte[] toBytes() throws IOException {

		Player player = new Player(101, 20, "peter", "男");
		player.getSkills().add(1001);

		ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
		ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);

		// 写入对象
		objectOutputStream.writeObject(player);

		// 获取字节数组
		byte[] byteArray = byteArrayOutputStream.toByteArray();
		return byteArray;
	}

	/**
	 * 反序列化
	 */
	public static void toPlayer(byte[] bs) throws Exception {

		ObjectInputStream inputStream = new ObjectInputStream(new ByteArrayInputStream(bs));
		Player player = (Player) inputStream.readObject();

		// 打印
		System.out.println("反序列化之后的内容...");
		System.out.println("playerId:" + player.getPlayerId());
		System.out.println("age:" + player.getAge());
		System.out.println("name:" + player.getName());
		System.out.println("skills:" + (Arrays.toString(player.getSkills().toArray())));
		System.out.println("gender:" + player.getGender());
	}

}
```

```
反序列化之后的内容...
playerId:101
age:20
name:peter
skills:[1001]
gender:null
```



# 5、serialVersionUID的作用

​	serialVersionUID字面意思上是序列化的版本号，凡是实现Serializable接口的类都有一个表示序列化版本标识符的静态变量 。

​	如果在源代码中不指定serialVersionUID，则在编译过程中，会根据类的内部细节自动生成一个serialVersionUID。如果对类的源代码作了修改，再重新编译，新生成的类文件的serialVersionUID的取值有可能也会发生变化。

​	对于同一个类，用不同的Java编译器编译，有可能会导致不同的 serialVersionUID。所以为了提高serialVersionUID的独立性和确定性，强烈建议在一个可序列化类中显示的定义serialVersionUID，为它赋予明确的值。

​	如果serialVersionUID不同，则在序列化与反序列的过程中就会出现问题。

​	显式地定义serialVersionUID有两种用途：

​	1）在某些场合，希望类的不同版本对序列化兼容，因此需要确保类的不同版本具有相同的serialVersionUID；

​	2）在某些场合，不希望类的不同版本对序列化兼容，因此需要确保类的不同版本具有不同的serialVersionUID。

通过如下代码演示，即可说明上边描述的内容：

1. 首先对没有明确标识serialVersionUID的Customer对象进行序列化与反序列化

   ```java
   package io.github.lin38.serialize.demo004;
   
   import java.io.Serializable;
   
   public class Customer implements Serializable {
   	// Customer类中没有定义serialVersionUID
   	private String name;
   	private int age;
   
   	public Customer(String name, int age) {
   		this.name = name;
   		this.age = age;
   	}
   
   	@Override
   	public String toString() {
   		return "name=" + name + ", age=" + age;
   	}
   
   	// 省略getter、setter
   	
   }
   ```

   ```java
   package io.github.lin38.serialize.demo004;
   
   import java.io.File;
   import java.io.FileInputStream;
   import java.io.FileNotFoundException;
   import java.io.FileOutputStream;
   import java.io.IOException;
   import java.io.ObjectInputStream;
   import java.io.ObjectOutputStream;
   
   public class TestSerialversionUID {
   
   	public static void main(String[] args) throws Exception {
   		serializeCustomer();// 序列化Customer对象
   		Customer customer = deserializeCustomer();// 反序列Customer对象
   		System.out.println(customer);
   	}
   
   	private static void serializeCustomer() throws FileNotFoundException, IOException {
   		Customer customer = new Customer("gacl", 25);
   		ObjectOutputStream oo = new ObjectOutputStream(new FileOutputStream(new File("F:/customer.data")));
   		oo.writeObject(customer);
   		System.out.println("Customer对象序列化成功！");
   		oo.close();
   	}
   
   	private static Customer deserializeCustomer() throws Exception, IOException {
   		ObjectInputStream ois = new ObjectInputStream(new FileInputStream(new File("F:/Customer.data")));
   		Customer customer = (Customer) ois.readObject();
   		System.out.println("Customer对象反序列化成功！");
   		ois.close();
   		return customer;
   	}
   }
   ```

   执行结果（反序列化成功）：

   ```
   Customer对象序列化成功！
   Customer对象反序列化成功！
   name=gacl, age=25
   ```

   

2. 修改Customer类，比如增加一个字段，再直接执行反序列化操作，即从上一步执行结果中的data文件直接反序列化

   执行结果（序列化失败，两次编译生成的serialVersionUID不同）：

   ```java
   Exception in thread "main" java.io.InvalidClassException: io.github.lin38.serialize.demo004.Customer; local class incompatible: stream classdesc serialVersionUID = -3492424122446889291, local class serialVersionUID = -4974496315957872982
   	at java.io.ObjectStreamClass.initNonProxy(ObjectStreamClass.java:616)
   	at java.io.ObjectInputStream.readNonProxyDesc(ObjectInputStream.java:1843)
   	at java.io.ObjectInputStream.readClassDesc(ObjectInputStream.java:1713)
   	at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:2000)
   	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1535)
   	at java.io.ObjectInputStream.readObject(ObjectInputStream.java:422)
   	at io.github.lin38.serialize.demo004.TestSerialversionUID.deserializeCustomer(TestSerialversionUID.java:29)
   	at io.github.lin38.serialize.demo004.TestSerialversionUID.main(TestSerialversionUID.java:15)
   ```

   

3. 给Customer类指定serialVersionUID，删除第一步中生成的data文件，重新执行上两步，发现程序不会报错了



# 6、Externalizable

​	Externalizable接口extends Serializable接口，而且在其基础上增加了两个方法：writeExternal(ObjectOutput out)和readExternal(ObjectInput in)。这两个方法会在序列化和反序列化还原的过程中被自动调用，以便执行一些特殊的操作。

​	使用Externalizable进行序列化，当读取对象时，会调用被序列化类的无参构造器去创建一个新的对象，然后再将被保存对象的字段的值分别填充到新对象中。这就是为什么输出结果中会显示调动了无参构造器。由于这个原因，实现Externalizable接口的类必须要提供一个无参的构造器，且它的访问权限为public。

<center>示例：</center>

```java
package io.github.lin38.serialize.demo005;

import java.io.Externalizable;
import java.io.IOException;
import java.io.ObjectInput;
import java.io.ObjectOutput;
import java.util.ArrayList;
import java.util.List;

public class Player implements Externalizable {

	private static final long serialVersionUID = -5248069984631225347L;

	private long playerId;

	private int age;

	private String name;

	private List<Integer> skills = new ArrayList<>();

	// 必须包含public类型的无参构造器
	public Player() {
		System.out.println("执行默认构造器");
	}
	
	public Player(long playerId, int age, String name) {
		this.playerId = playerId;
		this.age = age;
		this.name = name;
	}

	@Override
	public void writeExternal(ObjectOutput out) throws IOException {
		out.writeObject(this.playerId);
		out.writeObject(this.age);
		out.writeObject(this.name);
		out.writeObject(this.skills);
	}

	@Override
	public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
		this.playerId = (long) in.readObject();
		this.age = (int) in.readObject();
		this.name = (String) in.readObject();
		this.skills = (List<Integer>) in.readObject();
	}
	
	// 省略getter、setter
}

```

```java
package io.github.lin38.serialize.demo005;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.util.Arrays;

/**
 * Externalizable方式实现序列化与反序列化-写入数组
 */
public class JAVA2Bytes {

	public static void main(String[] args) throws Exception {
		byte[] bytes = toBytes();
		toPlayer(bytes);
	}

	/**
	 * 序列化
	 */
	public static byte[] toBytes() throws IOException {

		Player player = new Player(101, 20, "peter");
		player.getSkills().add(1001);

		ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
		ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);

		// 写入对象
		objectOutputStream.writeObject(player);

		// 获取字节数组
		byte[] byteArray = byteArrayOutputStream.toByteArray();
//		System.out.println("序列化之后的内容...");
//		System.out.println(Arrays.toString(byteArray));
		return byteArray;
	}

	/**
	 * 反序列化
	 */
	public static void toPlayer(byte[] bs) throws Exception {

		ObjectInputStream inputStream = new ObjectInputStream(new ByteArrayInputStream(bs));
		Player player = (Player) inputStream.readObject();

		// 打印
		System.out.println("反序列化之后的内容...");
		System.out.println("playerId:" + player.getPlayerId());
		System.out.println("age:" + player.getAge());
		System.out.println("name:" + player.getName());
		System.out.println("skills:" + (Arrays.toString(player.getSkills().toArray())));
	}
}
```

执行结果：

```
执行默认构造器
反序列化之后的内容...
playerId:101
age:20
name:peter
skills:[1001]
```

