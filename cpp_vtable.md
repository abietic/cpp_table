*virtual table group*

The primary virtual table for a class along with all of the associated secondary virtual tables for its **proper base classes**.

在这里推测proper base class是指除了primary base class以外的base class

*primary base class*

对于动态类(dynamic class)来说，一个唯一的基类（如果有基类），它共享在偏移量0处的虚指针。

*secondary virtual table*

一个基类的虚表实例，它被嵌入在一个继承了该基类的子类的虚表中

*inheritance graph*

一个图，节点表示一个类和它的所有子对象，边(arcs)连接着它的所有直接基类的节点。

*inheritance graph order*

一个在其继承图(inheritance graph)上从继承的最子类对象到基类对象进行深度优先遍历得到的顺序，其中：

* 没有节点被访问多于一次。（所以，一个虚基类子对象，和它的所有基类子对象，只会被访问一次。）

* 一个节点的子对象以其声明的顺序被访问。（所以，如果有代码 class A : public B, public C，A先被遍历，然后是B和它的子对象，最后是C和它的子对象。）

注意遍历可能是先序的或是后序的。除非特别说明，默认先序（子类先于他的基类）。

*empty class*

一个类没有除了empty data memebers的非静态数据成员，没有除了zero-width bit-fields以外的unnamed bit-fields，没有虚函数，没有虚基类，也没有非空非虚的更多基类(no non-empty non-virtual proper base classes)。

*empty data member*

在empty class类型中的一个可能重叠的非静态成员变量。（可能指为了使生成的对象有地址分配的？）

*morally virtual*

一个子对象X是Y的一个morally virtual基类，如果X要么是Y的一个虚基类，要么是Y的虚基类的一个直接或间接的基类。

*nearly empty class*

一个类含有一个虚指针，但是没有除了（可能的）虚基类以外的其他数据：

* 没有除了zero-width bitfields的非静态数据成员

* 没有不是empty、nearly empty或者virtual的直接基类

* 最多只能有一个非虚的，nearly empty的基类，

* 并且没有既是empty，又不是morally virtual，并且在类中的偏移量不是0的proper base class 

这样的类（指nearly empty class）即使是虚基类的时候也可能是primary base class，和子类共享虚指针。

# Virtual Table Layout

## General

一个虚表(*virtual table*; *vtable*)是被用来分发(dispatch)虚函数(virtual function)、访问虚基类子对象(access virtual base class subobjects)和运行时类型信息(information for runtime type identification; RTTI)的。每个拥有虚拟成员函数或是虚基类的类都有一系列相关的虚表。如果一个类被用来做其他类的基类了，那它可能有多个对应的虚表。
然而一个在继承最末端的类(most-derived class)的所有对象实例的虚表指针指向的都是同一套虚表。

一个虚表由一系列偏移量，数据指针和函数指针,以及由这些项目组成的结构构成。

Virtual call (vcall) offsets 用来为虚函数做指针调整，这些虚函数是在虚基类或是其子对象(subobject)中声明的，并且有继承它们的子类覆写了这些虚函数。这些项(entries)被分配在给虚基类使用的虚表当中，这些虚基类是

含有已被覆写的虚函数声明的基类

它们是用来寻找从虚基类到含有覆写函数(overrider)的衍生类(derived class)的必要的调整（如果有这样的衍生类）。

当一个虚函数通过一个虚基类被调用，但是这个虚函数在一个衍生类中被覆写了，这个覆写了它的函数(overriding function)首先将一个固定的偏移量


### Category 3: Virtual Base Only 只有虚基类的子类

结构：
* 只有虚基类（但是这些虚基类可以有非虚基类）。
* 这些虚基类既不是empty的也不是nearly empty的。

这个类对每个有虚表的虚基类都备有一个虚表。这些虚表都是secondary virtual tables，因为这些虚基类没有empty的或nearly empty的来做primary base class，并且这些虚基类的虚表是通过这些基类的完整对象虚表（根据category 2的规则）的副本(copies of the base class full object virtual tables)构造的，除了虚基类A的虚表还包含给每个在A的primary virtual table中出现的虚函数一个vcall offset项和来自A的非虚基类的secondary virtual table。

在虚基类A的secondary virtual table中的vcall offset的顺序解释如下。我们解释其排列顺序为，从项最接近vptr指向地址到最远。由于vcall offsets在vptr指向地址之前，这意味着内存地址顺序正好与vcall offset项排列顺序正好相反。

* 如果虚基类A有一个virtual base class P与其公用一个虚表，P的vcall offsets放在前面，这些项的顺序和P自己是虚基类时项的排列顺序一致。
* 接下来根据A中声明的虚函数的声明顺序给出接下来的vcall offsets。注意，即使是一个拥有不同类型返回值的覆写虚函数，也只有一个vcall offset（虽然会有多个虚表项），因为这个vcall offset可以被所有这样的虚表项共有。
* 最后是除了P以外的A的非虚基类所声明的虚函数。这些基类根据inhabitance graph的先序顺序排列vcall offsets，每个类中依然按虚函数的声明顺序排列。

如果上面列出的vcall offsets 包含超过一个特定的虚函数签名(includes more than one of a particular virtual function signature)，只有第一个（离vptr指向地址最近的那个）进行分配。也就是说，一个来自primary base P（和其非虚基类）的offset会消除来自A或A的其它基类的offsets，一个来自A的offset会消除所有来自其它non-primary的offsets，而一个来自A的non-primary base B的offset会消除B的基类的offsets。

*注意，对于A的虚基类V中声明的所有没有被A及A的非虚基类覆写的虚函数，这些虚函数在A的vcall offsets中没有对应。对这些函数的调用会使用V的虚表中的vcall offset。*

这个类还有一个不是从虚基类的虚表复制而来的虚表。这个虚表是类的主要虚表(primary virtual table)，它是通过对象顶端的vptr寻址的，这个vptr并不是和其它基类共享的，因为结构规定没有nearly empty的虚基类来作为primary base class。这个虚表存储该类的所有虚函数指针项。因为没有primary base class，这个虚表的虚函数指针项中包含了所有覆写了基类的函数的项；这些项又被称作复制项(replicated entries)因为它们已经在类的次要虚表(secondary virtual table)中了。

主要虚表还包含虚基类偏移量，来保证找到虚基类子对象。不管是直接还是间接的虚基类都拥有一个对应的虚基类偏移量表项。这些表项都是以继承图顺序(inheritance graph order)的逆序排列。也就是说，最左侧的虚基类偏移量表项离vptr指向的地址最近。

### Cateory 4: Complex

结构：
* 不是上述中的任一种，也就是说直接或间接地继承虚拟或非虚的多个基类(proper base class)，或是至少有一个nearly empty的虚基类。

构造虚表的规则是类别2和类别3的组合，并且基本可以通过归纳法确定。区别主要是由于现在虚基类（在nearly empty的情况下）现在可以作primary base了。

* 如果虚基类A有一个primary的虚基类P共享虚表，P的vbase和vcall偏移量先在主要虚表(primary virtual table)中出现，出现的顺序和P自己作虚基类时相同，而来自A的表项不会重复P之前已经给出的。
* 而对于非虚基类的情况，衍生类的虚函数指针表项（就像上面所说的一样）出现在主要基类虚函数表项的下面。

```c++
/*
Test case for sharing virtual bases.
In Derived_too,
the primary base class is NewShareme,
The bases Base and Shareme share vptr's
with Derived and are allocated at the
same offset as Derived.
Should get:
60% a.out
(long)(NewShareme *)dd - (long)dd = 0
(long)(Derived *)dd - (long)dd = 8
(long)(Base *)dd - (long)dd = 8
(long)(Shareme *)dd - (long)dd = 8
*/

struct Shareme {
    virtual void foo();
};
struct Base : virtual Shareme {
        virtual void bar();
};
struct Derived : virtual Base {
        virtual void baz();
};

struct NewShareme {
        virtual void foo();
};

struct Derived_too : virtual NewShareme, virtual Derived {
        virtual void bar();
};

void Shareme::foo() { }
void Base::bar() { }
void Derived::baz() { }
void NewShareme::foo() { }
void Derived_too::bar() { }


extern "C" int printf(const char *,...);
#define EVAL(EXPR) printf( #EXPR " = %d\n", (EXPR) );
main()
{
  Derived_too *dd = new Derived_too;
  EVAL((long)(NewShareme *)dd - (long)dd);
  EVAL((long)(Derived *)dd - (long)dd);
  EVAL((long)(Base *)dd - (long)dd);
  EVAL((long)(Shareme *)dd - (long)dd);
}
```

```c++
/*
Test case for sharing virtual bases.
In Most_Derived,
the primary base class is Nonvirt1,
Nonvirt2 and Nonvirt3 share vptrs with
virtual base Shared_Virt.  Shared_Virt
should be at the same offset as Nonvirt2.
Should get:
67% a.out
(long)(Nonvirt1 *)dd - (long)dd = 0
(long)(Nonvirt2 *)dd - (long)dd = 8
(long)(Nonvirt3 *)dd - (long)dd = 16
(long)(Shared_Virt *)dd - (long)dd = 8
*/

struct Shared_Virt {
    virtual void foo();
};
struct Nonvirt2 : virtual Shared_Virt {
        virtual void bar();
};
struct Nonvirt3 : virtual Shared_Virt {
        virtual void baz();
};
struct Nonvirt1 {
        virtual void foo();
};

struct Most_Derived : Nonvirt1, Nonvirt2, Nonvirt3 {
        virtual void bar();
};

void Shared_Virt::foo() { }
void Nonvirt2::bar() { }
void Nonvirt3::baz() { }
void Nonvirt1::foo() { }
void Most_Derived::bar() { }

extern "C" int printf(const char *,...);
#define EVAL(EXPR) printf( #EXPR " = %d\n", (EXPR) );
main()
{
  Most_Derived *dd = new Most_Derived;
  EVAL((long)(Nonvirt1 *)dd - (long)dd);
  EVAL((long)(Nonvirt2 *)dd - (long)dd);
  EVAL((long)(Nonvirt3 *)dd - (long)dd);
  EVAL((long)(Shared_Virt *)dd - (long)dd);
}
```

```c++
/*
Test case for sharing virtual bases.
In Most_Derived, share the vptr with
Interface1 but not Interface3, since
Interface3 is indirectly inherited.

Should get:
(long)(Interface1 *)dd - (long)dd = 0
(long)(Interface2 *)dd - (long)dd = 8
(long)(Interface3 *)dd - (long)dd = 8
(long)(Concrete1 *)dd - (long)dd = 8
*/

struct Interface1 {
    virtual void foo();
};
struct Interface2 : virtual Interface1 {
        virtual void bar();
};
struct Interface3 : virtual Interface2 {
        virtual void baz();
};

struct Concrete1 : virtual Interface3 {
        virtual void foo();
        int i; // important.
};

struct Most_Derived : virtual Interface1, 
                      virtual Interface2,
                      virtual Concrete1 {
        virtual void bar();
};

void Interface1::foo() { }
void Interface2::bar() { }
void Interface3::baz() { }
void Concrete1::foo() { }
void Most_Derived::bar() { }


extern "C" int printf(const char *,...);
#define EVAL(EXPR) printf( #EXPR " = %d\n", (EXPR) );
main()
{
  Most_Derived *dd = new Most_Derived;
  EVAL((long)(Interface1 *)dd - (long)dd);
  EVAL((long)(Interface2 *)dd - (long)dd);
  EVAL((long)(Interface3 *)dd - (long)dd);
  EVAL((long)(Concrete1 *)dd - (long)dd);
}
```

## 2.6 Virtul tables During Object Constuction 对象构造过程中使用的虚表

### 2.6.1 General

在一些情况下，一个叫做构造虚表(construction virtual table)的特殊虚表在多余基类(proper base class)的构造函数和虚构函数执行时使用。这些虚函数是为了应对虚继承时的特殊情况。

在构造一个类对象的过程中，对象在它的每个基类(proper base class)的子对象构造时假定(assumes)它们的类型。RTTI要求在基类的构造函数中要返回基类的类型，并且虚拟调用(virtual calls)会会指向基类的成员函数而非完整类(complete class)的。RTTI要求，在构造过程中对于对象的动态类型转换(dynamic casts)和虚拟调用要静态地转换成对于正在进行构造的基类和动态地将调用指向正在构造中的基类函数。通常来说，这种行为都是通过在基类的构造函数中通过设置对象的虚表指针指向基类的虚表地址来完成的。

然而，如果基类是一个直接或间接的虚基类，虚表指针就必须设置指向构造虚表的地址。这是因为普通的基类虚表可能没有存储正确的虚基类索引值来访问正在构造中的对象的虚基类，并且如果在进行从一个虚基类向对象的另一部分转换时this指针参数的调整可能因为使用这样的虚表而调整出错。问题出在一个基类(proper base class)的完整对象和一个派生类的完整对象虚基类所在位置的偏移量不同。

一个构造虚表保存虚函数地址、到对象顶端的偏移量、与基类相关的RTTI信息、虚基类的偏移量和调整器的入口地址以及其与完整类对象相关的参数调整(addresses of adjustor entry points with their parameter adjustments associated with objects of the complete class)。

为了保证虚表指针在基类(proper base class)构造时被设置为指向合适的虚表，一个关于虚表指针的表，被称作VTT，其存储着为一个完整类生成的用于构造和不用于构造的虚表的地址。完整类的构造函数为每个基类(proper base class)的构造函数传递一个指向VTT合适位置的指针，在这个位置基类(proper base class)的构造函数可以找到其所需的一组虚表。构造虚表在执行基类(proper base class)的析构函数时也以类似的方式使用。

当一个完整对象构造函数正在构造一个虚基时，它必须小心使用虚表中的虚基类偏移量(vbase offsets)，因为可能的共享vptr可能指向一个无关基类的构造虚表，例如在

```c++
struct S {};
struct T: virtual S {};
struct U {};
struct V: virtual T, virtual U {};
```

中，V和T的vptr在同一位置。当V的构造函数要构造U时，这时这个vptr指向的是T的虚表，因此不能用来定位U。（一般完整类的构造函数直接静态确定不久行了？）

### 2.6.2 VTT Order VTT顺序

虚表地址的数组，被称作***VTT***，是为每个有直接或间接虚基类的类型声明的。（否则，每个基类(proper base class)可以通过完整类的虚表组(complete object table group)进行初始化。）

类D的VTT数组的元素以如下顺序排列：

1. *Primary virtual pointer*