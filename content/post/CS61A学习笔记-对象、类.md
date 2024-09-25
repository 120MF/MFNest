+++
title = 'CS61A学习笔记 对象、类'
date = 2023-10-22T18:35:03+08:00
draft = false
description = ""
slug = "CS61A学习笔记-对象、类"
image = "/media/image-127.png"
categories = ["编程相关"]
tags = ["CS61A","Python","学习笔记","面向对象"]
weight = 1       # You can add weight to some posts to override the default sorting (date descending)
keywords = ["CS61A","Python","学习笔记","面向对象"]
readingTime = true
+++

想象我们用Python构建一个巧克力商店，我们可能会用这些函数建立抽象数据结构来储存各种信息：

<img src="/media/image-127.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

然而，我们也可以用对象和类来编写和维护相关代码。事实证明，这种面向对象式编程会提高我们的开发效率和能力。

<img src="/media/image-128.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

## 基本概念

在面向对象编程中，有一些常用的名词：

- 类（class）是一种模板，可以用类（class）来创建新的数据类型；
- 对象（object）是某个类的实例；
- 对象可以存储实例变量（instance variables），描述其状态和数据属性；
- 对象可以有方法（method），作为其相关的函数。

## 类（class）与对象（object）

编写下面的代码来新建一个类并构建其初始化代码。

<img src="/media/image-129.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

在这段代码中，__init__函数会在生成对象时自动执行。该函数的作用正是用来初始化一个对象/类。传入__init__中的第一个参数永远都是新对象的变量名。该形参名称的指定也是任意的（也可以改成this）。我们用"."表示访问对象中的属性，将"_"加在属性之前表示其为私有属性。（仅为约定俗称）

我们把运行上面代码的结果称为类实例化（class instantiation），我们在这个过程中构造了一个类（Object construction），Product(args)就是构造函数。在运行完构造函数之后，新对象的框架将被绑定到全局而非__init__函数的局部框架。

在类中，我们可以定义实例变量，还可以定义方法。比如：

<img src="/media/image-130.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

在这里，我们把increase_inventory()称为绑定方法（bound method），它将调用方法的对象绑定作为参数传入了函数中。

事实上，我们也可以这样写：

<img src="/media/image-131.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

可以从它们的类型上看出它们的区别。

<img src="/media/image-132.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

（补充：一种格式化字符串的方法）

<img src="/media/image-133.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

对于我们没有初始化的实例变量，我们也可以在之后的类操作过程中随时新建。

<img src="/media/image-134.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>新的_needs_restoking实例变量</em></p>

我们将直接存储在class内（而不是在方法中）的变量为类变量。（class variable）

<img src="/media/image-135.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

我们不能在类的实例（也就是对象）中直接操作类变量，我们应该通过类对类变量进行操作。

<img src="/media/image-136.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

在构造对象时，_sales_tax并没有在对象中被创建，所以用=来赋值时，python会在对象中新建一个变量，而不是对类变量进行修改。相对应的，当我们在实例中使用"."来查找该_sales_tax变量时会被返回全局的类变量（向上查找）。

## 继承（inheritance）

有时，两个类会出现高度相似的写法。为了避免代码重复，我们可以使用“继承”方法将两部分代码耦合在一起。

<img src="/media/image-137.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

我们要定义一个基类（bass class/superclass），在这个集类上定义子类。

<img src="/media/image-121-1024x663.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

我们首先编写Animal类的基础代码，然后在全局框架下将Animal类作为新定义的类的参数，就能创建出Animal的子类。

<img src="/media/image-138.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

在子类中我们也可以新建基类中不存在的实例变量。

<img src="/media/image-139.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

同样可以重写基类中的方法。

<img src="/media/image-126-1024x295.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

有时，我们需要访问基类中的实例变量或者方法，我们可以使用super()函数，它会返回当前对象的基类。这种做法通常出现在我们需要使用基类的功能，又需要添加子类的额外代码的时候。

<img src="/media/image-140.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>但是使用super获取到的方法的运行环境还是在当前类中</em></p>

## 多重继承

python3中的几乎所有东西都可以被看作对象，他们都由基类object继承而来。

<img src="/media/image-141.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<img src="/media/image-142.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>列表的基类是object</em></p>

我们也可以在此基础上设计出更复杂的继承关系：

<img src="/media/image-143.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<img src="/media/image-144.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>将食草动物作为Animal的一个子类</em></p>

<img src="/media/image-145.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>将具体的动物作为食草动物的子类</em></p>

我们还可以更进一步，让一个子类继承自多个基类：

<img src="/media/image-146.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<img src="/media/image-147.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<img src="/media/image-148.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>继承自多个基类</em></p>

这样确实可能在某些时刻会很方便，但是代价是重写和维护起来相当困难，要弄清楚继承关系会随着代码量的增大愈加棘手。

## 对象基类（object class）

之前我们提到，python3中的几乎所有东西都可以看作是对象，他们都继承自object类。我们可以看看object类中都有些什么方法。（使用dir()函数）

<img src="/media/image-149.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

python3拥有很多语法糖，这些糖大部分都依赖于object基类中的基本方法所做的幕后工作。比如下面的__str__方法：

<img src="/media/image-150.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<img src="/media/image-151.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>这些函数都在幕后调用了__str__方法</em></p>

由于我们自己创建的类也继承了object，因此我们可以在自己的类上重写__str__方法，进而更改其他函数对待我们自定义类的方法：

<img src="/media/image-152.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

与__str__类似的，我们有__repr__方法。不同的是，该方法直接返回原始的代码。我们可以配合eval()（执行参数内的代码）函数来使用__repr__方法。

<img src="/media/screenshot_20231024_1023077941129435639914478.jpg" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

事实上，在交互式环境中，当我们输入对象名时，屏幕上显示的就是对象名.__repr__()

我们也可以像对__str__一样重写__repr__：

<img src="/media/screenshot_20231024_1038041775123394365474965.jpg" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

对于上面所说的两种object方法，我们可以用下面任意一种方法来调用它们：

<img src="/media/screenshot_20231024_1030325746590489463367553.jpg" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

我们也可以直接使用repr()函数，该函数在幕后仍然会使用__repr__方法，但是这么做避免我们直接操作复杂的方法和逻辑。这也揭示了为什么__repr__要加双下划线——我们不应该也不需要经常来使用这些方法。

在大部分时候，我们使用"."来获得属性。然而，用这种方法访问不存在的属性时python会抛出异常。因此我们可以使用getattr()函数来获取对象的属性，该函数包含一个默认值，若属性不存在就会创造属性并且为其赋默认值。

<img src="/media/screenshot_20231024_1046274452245687022216096-1024x584.jpg" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

我们还可以用hasattr()函数检查对象中的属性，避免python抛出异常。

<img src="/media/image-142-1024x454.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

事实上，不管用何种方法，我们访问属性的幕后工作都是__getattribute__方法。

<img src="/media/image-153.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

object基类还有很多其它的特殊方法，我们可以在自己的类中重写它们，以自定义我们类的行为。

<img src="/media/image-143-1024x743.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">