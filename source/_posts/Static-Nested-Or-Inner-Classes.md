---
title: Static Nested Or Inner Classes
date: 2019-12-06 15:35:59
toc: true
tags:
 - Java
 - 最佳实践
categories: Java基础
---

在 Java 中，在一个类中声明另一个类则称为嵌套类，被声明为 `static` 的嵌套类称为静态嵌套类（`static nested classes` ），与之相对的非静态嵌套类被称为内部类(（`inner classes` ）

- 非静态嵌套类每个实例都包含一个额外指向外围对象的引用，换句话说，要实例化一个非静态嵌套类必须首先实例化外部类

- 静态嵌套类独立于外部类实例，可以看作嵌套在一个顶级类中的顶级类。因此，如果嵌套类不要求访问外部类的实例变量或方法，就要始终把 `static` 修饰符放在它的声明中，使它成为静态嵌套类。（如果该嵌套类不作为基类，那么更适合同时加上 `final` 修饰符）。JDK1.8 源码可见各种这样的设计，如 ReentrantLock 中

  ```java
  static final class NonfairSync extends Sync {
  	...
  }
  ```

我们从四个方面来更详细的讨论它们的区别：

- 嵌套类访问外部类的范围

- 嵌套类本身定义变量的范围

- 实例化

- 同名覆盖

<!-- more -->

# 非静态嵌套类

- 非静态嵌套类和外部类的实例关联，可以直接访问外部类的所有方法和字段

  ```java
  public class OuterClass {
  
      private static int i = 1;
      private int j = 2;
  
      public void display(){
          System.out.println("OuterClass...");
      }
  
      class InnerClass {
  
          public void run(){
              System.out.println(i);
              System.out.println(j);
              display();
          }
      }
  }
  ```

- 同样由于非静态嵌套类和外部类的实例相关联，所以它不能自己定义任何静态成员

  ```java
  public class OuterClass {
  
      class InnerClass {
  
          static int failedField = 1;  // 编译错误
        
          static void failedMethod(){  // 编译错误
  
          }
      }
  }
  ```

- 需要先实例化外部类，再实例化非静态嵌套类

  ```java
  OuterClass outerClass = new OuterClass();
  OuterClass.InnerClass innerClass = outerClass.new InnerClass();
  ```

- 同名覆盖问题，非静态嵌套类仅会出现在实例变量或方法中，非静态嵌套类中声明的实例同名变量或方法会覆盖外部类的声明，访问外部类的实例变量或方法需要加上 `外部类名.this`，如 `OuterClass.this.x`

  ```java
  public class OuterClass {
  
      private int j = 2;
  
      public void display(){
          System.out.println("OuterClass...");
      }
  
      class InnerClass {
  
          private int j = 3;
  
          public void run(){
              System.out.println(this.j);  // 3
              System.out.println(OuterClass.this.j); // 2
              this.display();  // InnerClass..
              OuterClass.this.display();  // OuterClass...
          }
  
          public void display(){
              System.out.println("InnerClass...");
          }
      }
  }
  
  ```

# 静态嵌套类

- 静态嵌套类和外部类（非实例）相关联，因此仅能访问外部类的静态变量和方法

  ```java
  public class OuterClass {
  
      private static int staticField = 1;
      private int normalField = 2;
  
      public static void staticMethod(){
          
      }
      
      public void normalMethod() {
          
      }
    
      static class StaticNestedClass {
        
      	public void display() {
            System.out.println(staticField);  // 编译通过
            System.out.println(normalField);  // 编译错误
            staticMethod();  // 编译通过
            normalMethod();  // 编译错误
        }
      
  }
  ```

- 由于静态嵌套类不依赖于外部类实例，所以它可以定义任意变量和方法（和普通类相同）

  ```java
  public class OuterClass {
        
        static class StaticNestedClass {
            
            int i = 1;
            static int j = 2;
  
            public void normalDisplay() {
                System.out.println(i);
                System.out.println(j);
            }
            
            public void staticDisplay(){
                System.out.println(j);
            }
        }
    }
  ```


- 可以直接实例化静态嵌套类，且不会实例化外部类

  ```java
  OuterClass.StaticNestedClass staticNestedClass = new OuterClass.StaticNestedClass();
  ```

- 同名覆盖问题，静态嵌套类仅会出现静态变量或方法重名，静态嵌套类中声明的静态同名变量或方法会覆盖外部类的声明，访问外部类的静态变量或方法需要加上外部类名，如 `OuterClass.x`

  ```java
  public class OuterClass {
  
      static int j = 4;
  
      static class StaticNestedClass {
  
          static int j = 2;
  
          public void display() {
              System.out.println(i);  // 1
              System.out.println(j);  // 2
            	System.out.println(OuterClass.j);  // 4
          }
      }
  }
  ```

最后关于序列化，根据 Oracle 官方建议，强烈不推荐序列化非静态嵌套类，原因参考 [Nested Classes](https://docs.oracle.com/javase/tutorial/java/javaOO/nested.html)

# 应用场景

当一个类的构造函数需要传入多个参数，且很多参数是可选的，传统做法是重载构造函数。这将导致几个问题

- 由于可选参数多，因此会有大量重载构造函数，也就是说会存在大量重复代码
- 客户端调用比较困难，使用者需要选择合适的构造函数，以及了解每个参数的含义和顺序

因此比较合适的方式是使用 `Builder` 模式代替重载构造函数，它隐藏了内部的具体构建细节，允许多个可选参数，具有很强的可读性。而 `Builder` 模式就是使用静态嵌套类实现的。

```java
@Getter
public class BaseVideoParseBo {

    /**
     * 电影
     */
    private String movie;

    /**
     * 搜索链接
     */
    private String searchUrl;

    /**
     * 电影标题
     */
    private String title;

    /**
     * 时长
     */
    private String time;

    /**
     * 清晰度
     */
    private String bagde;

    /**
     * 封面
     */
    private String imgUrl;

    /**
     * 视频链接
     */
    private List<String> videoUrls = new ArrayList<>();

    /**
     * 渠道
     */
    private String channel;

    protected BaseVideoParseBo(Builder<?> builder) {
        this.movie = builder.movie;
        this.bagde = builder.bagde;
        this.searchUrl = builder.searchUrl;
        this.time = builder.time;
        this.title = builder.title;
        this.imgUrl = builder.imgUrl;
        this.channel = builder.channel;
    }

    public static class Builder<T extends Builder<T>> {

        private Class<T> subBuilder;
        private String movie;
        private String searchUrl;
        private String title;
        private String time;
        private String bagde;
        private String imgUrl;
        private String channel;

        public Builder(Class<T> subBuilder) {
            this.subBuilder = subBuilder;
        }

        public T movie(String movie) {
            this.movie = movie;
            return self();
        }

        public T searchUrl(String searchUrl) {
            this.searchUrl = searchUrl;
            return self();
        }

        public T title(String title) {
            this.title = title;
            return self();
        }

        public T time(String time) {
            this.time = time;
            return self();
        }

        public T bagde(String bagde) {
            this.bagde = bagde;
            return self();
        }

        public T imgUrl(String imgUrl) {
            this.imgUrl = imgUrl;
            return self();
        }

        public T channel(String channel) {
            this.channel = channel;
            return self();
        }

        private T self() {
            return subBuilder.cast(this);
        }

        public BaseVideoParseBo build() {
            return new BaseVideoParseBo(this);
        }
    }

    public void addVideoUrls(List<String> videoUrls) {
        this.videoUrls.addAll(videoUrls);
    }

    public void addVideoUrl(String videoUrl) {
        this.videoUrls.add(videoUrl);
    }
}
```

构建该类对象代码如下：

```java
// 仅使用基类时
BaseVideoParseBo baseVideoParseBo = new BaseVideoParseBo.Builder<>(BaseVideoParseBo.Builder.class)
				.xxx()
				.build();
```

这里再做一些拓展，可以看到上述类大量使用了泛型，其主要用于解决 `Builder` 继承时的返回类型问题，因此结合了 `Builder` 模式和协变返回类型，其子类可以如下实现：

```java
@Getter
public class SubParseBo extends BaseVideoParseBo {

    /**
     * 是否获取所有清晰度的视频
     */
    private boolean allVideos;

    private String dataId;
    private String dataInfo;
    private String videoId;
    
    protected SubParseBo(Builder builder) {
        super(builder);
        this.dataId = builder.dataId;
        this.allVideos = builder.allVideos;
    }

    public static class Builder extends BaseVideoParseBo.Builder<Builder> {

        private boolean allVideos;
        private String dataId;

        public Builder() {
            super(Builder.class);
        }

        public Builder dataId(String dataId) {
            this.dataId = dataId;
            return this;
        }

        public Builder allVideos(boolean allVideos) {
            this.allVideos = allVideos;
            return this;
        }

        @Override
        public SubParseBo build() {
            return new SubParseBo(this);
        }
    }

    public void setDataInfo(String dataInfo) {
        this.dataInfo = dataInfo;
    }

    public void setVideoId(String videoId) {
        this.videoId = videoId;
    }
}
```

那么构建子类对象如下代码：

```java
SubParseBo parseBo = new SubParseBo.Builder()
                .xxx
                .build();
```

