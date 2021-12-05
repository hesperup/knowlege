SOLID五大原则：单一职责原则，开闭原则，里氏替换原则，依赖倒转原则和接口隔离原则。



**Advice 1：**

**When specifying a collection of data, use abstract classes for data types and extended classes for variants.**

OO是不喜欢函数指针的，继承和多态可以消除函数指针，实现动态Dispatch。同时抽象类可以让模块之间依赖减少，多种不同的派生类负责具体的业务逻辑。这里讲的应该是 datatype的可自由组合，你可以把datatype跟乐高作对比，每一个datatype就是乐高积木的一个形状。



**Advice 2：**

**When writing a function over a datatype, place a method in each of the variants that make up the datatype. If a field of a variant belongs to the same datatype, the method may call the corresponding method of the field in computing the function.**

里氏替换原则里面要求，派生类必须完整实现父类定义的接口，否则父类和派生类就不是可以随意替换的。如果派生类的数据成员被其他派生类所共有，通常情况下会在实现子类方法的时候调用父类的实现。（即调用super.methodXXX())。这一层，讲的是datatype的可替换性，这个类似乐高里面的点槽，形状一样，点槽不一样的积木是没办法自由组合的。



**Advice 3：**

**When writing a function that returns values of a datatype, use new to create these values.**

这里就不再是简单的OO原则了，而是FP原则。尽量减少副作用（Side Effect），同时返回新的对象还可以实现链式调用，增强OO函数的可组合性。函数的参数是datatype，返回值也是datatype，这样就可以自由组合了，达到最大限度的灵活性（注意：datatype必须是抽象的）



**Advice 4:**

**When writing several functions for the same self-referential datatype, use visitor protocols so that all methods for a function can be found in a single class.**

通常，我们会选择直接给一个类添加方法来进行扩展。如果业务有变化，我们就新建子类并在子类中覆盖父类的实现来完成功能扩展和定制。但是，如果我们把某一个方法抽象成一个类，然后该类型子类所有的实现都可以聚集在一个class中。因为类型是可以参数化的（还记得吗？多态），所以，原本的方法抽象成类以后，我们就拥有了更强大的可组合性。另外，再多提一点，把所有子类型判断是不是Vegetarian都放在一个类中，其实也有利于功能内聚，很多OO模式（状态模式，命令模式，策略模式）关注的也是用Class去封装行为，而不只是封装数据。



**Advice 5**

**When the additional consumed value change for a self-referential use of a visitor, don't forget to create a new visitor.**

每一组方法（这组方法是为该datatype所有的派生类准备的）抽象成一个类以后，这些类可以组成一个类的继承结构，每一个派生类都是一个Visitor。



**Advice 6:**

**When designing visitor protocols for many different types, create a unifying protocol using Object.**

当为许多不同类型设计 Visitor时，尽可能统一协议，即采用Object作为每个Visitor方法的参数。因为Java里面所有的对象都是从Object派生过来的，这样子设计的visitor实际上扩展性是更强了。但是，这也会遇到一些向下转型（downcasting)的问题，需要使用运行时的类型信息做一些额外的处理。



**Advice 7：**

**When extending a class, use overriding to enchrich its functionality.**

当扩展一个类的时候，使用重载（overriding) 的方式来增强其功能。这里其实也是呼应前面提到的抽象类和子类的概念，利用OO的多态实现功能扩展。OO hierarchy between datatypes 和 OO hierarchy between datatype's method groups，这里visitor类，更像是一个个高度抽象化的函数，对比函数式编程（FP），函数是可以自由组合的，只要签名一样，函数还可以自由替换。



**Advice 8:**

**If a datatype many have to be extended, be forward looking and use a constructor-like (overridable) method so that visitors can be extended, too.**

如果一个数据类型需要被扩展，那么也需要同时考虑visitor是否可以兼容这种扩展。最好是使用可以被override的方法，这样visitor的子类型就可以覆盖这些方法。



**Advice 9:**

**When modifications to objects are needed, use a class to insulate the operations that modify objects. Otherwise, beware the consequences of your actions.**

当必须要对一个对象进行修改的时候，不要每次都直接在该对象的类型上添加一个方法和若干数据成员来解决。可以考虑，是否可以通过添加一个visitor class来解决。因为visitor的方式可扩展性更强，如果直接添加方法可以解决问题，并且后续需求变化不需要频繁对该方法进行修改，那么就没有必要引入visitor，反之，则需要使用visitor来解耦代码。