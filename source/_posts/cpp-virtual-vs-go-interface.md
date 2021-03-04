---
title: Go interface internal
date: 2018-05-05 03:01:17
tags: 
    - go 
    - programing lanuage
---

运行期的多态，是很多高级语言的很重要的特性。我们可以在运行期根据实际创建的对象，决定实际所要运行的函数。
C++可以通过virtual member function实现运行期的多态，同样，在Go我们可以通过interface实现类似的机制。虽然C++的virtual和Go的interface本质上还是有很多区别，但是不妨碍我们深入探讨一下关于如何实现运行期多态。

> Note: 
> 1. 不去理解实现机制，并不影响我们使用C++或者Go（只要遵循语言的标准并不会有什么麻烦）。当然，如果能更深入的理解实现，本身有利于加深我们对代码的理解。
> 2. 我们讨论的的实现很可能只代表主流编译器的实现，并不一定代表语言的标准。编译器需要遵循语言规范去实现，但是如何实现往往是编译器自己的决定。

## C++多态 

我们都知道C++的多态，通过基类里面把成员函数定义为virtual，继承类可以重新实现这个函数。使用一个基类的指针指向对象，并且通过该指针执行那个函数，具体执行的是基类还是继承类的函数，取决于运行期该对象的类型。C++的多态/继承当然没有那么简单，还有非常多的细节特性，想要完全说明白几乎不可能（建议去翻C++标准或者经典类的书）。我们这里只需要用最简单的例子，去说明实现的原理。

例如下面的代码：

```
class Base {
public:
    virtual void func() { std::cout << "Hello Base" << std::endl; }
}

class D : public Base {
public:
    virtual void func() { std::cout << "Hello D" << std::endl; }
}

/// main.cpp
Base *p = new D();
p->func(); ```

由于指针p指向的对象，实际上是类型D。而且`func()`定义为`virtual`，所以实际执行的是D的`func()`，程序输出：

```
Hello D
```

## C++ vtable

C++通过vtable实现多态，C++在***编译期***给每个类生成一个vtable。vtable就是一个函数地址数组，记录了这个类所有的virtual function指向的函数地址。只要该类声明了function为virtual，或者它的父类/祖先类声明了virtual，则该function就是virtual。

例如，`class D`的vtable只包涵一个元素，就是`func`的地址。

```

class D : vtable --> -----------
                     | D::func |
                     -----------
                     |  null   |(null for array end)
                     -----------


```

这里需要注意，`class D`的`func`没有指向`Base::func`，而是指向了`D::func`，是由于`class D`重新实现了`virtual func`。如果`class D`没有实现`virtual func`，那么`class D`的`func`会指向`Base::func`。

```

class D : vtable --> --------------
                     | Base::func |
                     --------------
                     |  null      |(null for array end)
                     --------------


```

那么，vtable是怎么使用的呢。在我们创建对象的时候，编译器会在创建对象的内存的开始插入vtable的地址指针vptr(很tricky)。例如，我们`new D()`

```
                                           vtable
p = new D() -----> ------------          -----------
                   |  vptr    | -------->| D::func |
                   ------------          -----------
                   |          |          |  null   |
                   | D object |          -----------
                   |          |
                   ------------
```

> 所以即使是一个空的class，如果有virtual function，那么该class的sizeof也不会是0，而是一个指针的大小(sizeof(void *))。

当我们通过该对象调用`func`的时候，因为该function为virtual，会通过`vptr`找到`vtable`，并且在`vtable`里面找到`func`的地址，并且执行。

C++通过vtable实现了运行期的多态，但是vtable是在编译器静态绑定，运行期需要进行vtable的查找。


## Go interface
Go的interface是Go语言一个很重要的设计，借鉴了Java和C++的部分语言特性，最大的改变是去掉了C++和Java里面的显示继承。interface只是单纯的定义分行为（方法），如果我们要定义一种类型属于该interface，并不需要显示的继承该interface，只需要对该类型实现所有interface所声明的方法，那么这个类型就是属于该interface。

这里涉及的设计哲学是`What I do makes who I am`，不同于C++/Java的`I declare who I am`。这是一种充分`解耦`的设计哲学，打破 了类型之间的强耦合。其实跟现实世界很像，你是什么样的人，不取决于你自己怎么说或者你身上的既定的标签，而取决于你自己的行为。例如，你成天跟大家说你是君子不代表你就是一个君子，你的君子行径才能决定。嗯，有点跑题了，我们还是回到正轨。

例如下面的代码：

```
type Base interface {
    func()
}

type D struct {}

func (d *D) func() {
    fmt.Println("Hello D")
}

/// main.go
var p Base = new(D)
p.func()
```

由于p指向的对象，实际上是类型D，所以最后执行的是D的`func()`，程序输出：

```
Hello D
```


## Go itable

Go的interface结构包含2部分，第一部分是指向实际对象的value，第二部分是一个地址数组`itable`（有点像C++的vtable）。例如`interface Base`的结构如下

```

Base -->  -------------
          |   value   |
          -------------         ---------
          |  itable   | ----->  |  func |
          -------------         ---------
                                |  null |
                                ---------
```


当我们执行`var p Base = new(D)`时，会把左值拷贝给value，并且通过查找`D`的函数列表组装`itable`。

```

Base -->  -------------                                        -------------------
          |   value   |------------------------------------->  | address of D()  |
          -------------         ---------                      -------------------
          |  itable   | ----->  |  func |---------------
          -------------         ---------              |            -------------
                                |  null |              |----------> | D::func() |
                                ---------                           -------------

```

所以，当我们执行`p->func()`时，编译器已经知道具体执行的是`D::func()`


## 总结

C++和Go interface内部实现机制并不一样，但是都可以实现运行期的多态绑定。就是说，我们可以根据运行期实际创建的对象是什么，来决定运行的函数。

C++通过编译器静态产生vtable的方式实现，但是需要在运行期进行vtable查找，会带来运行期开销。

Go interface在运行期计算itable，好处是在执行的时候基本上是O(1)开销。
