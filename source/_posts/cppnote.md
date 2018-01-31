---
title: C and C++ notes
date: 2018-02-01 01:44:53
tags:
---
# C/C++

## Bit field
Bit Fields allow the packing of data in a structure. This is especially useful when memory or data storage is at a premium. Typical examples:
- Packing several objects into a machine word. e.g. 1 bit flags can be compacted -- Symbol tables in compilers.
- Reading external file formats -- non-standard file formats could be read in. E.g. 9 bit integers.
### Explanation[^1]
The number of bits in a bit field sets the limit to the range of values it can hold:
```c
struct S
{
// three-bit unsigned field,allowed values are 0...7
    unsigned int b : 3;
};
int main()
{
    S s = {7};
    ++s.b; // unsigned overflow (guaranteed wrap-around)
    std::cout << s.b << '\n'; // output: 0
}
```
Multiple adjacent bit fields are usually packed together (although this behavior is implementation-defined):
```c
struct S
{
    // will usually occupy 2 bytes:
    // 3 bits: value of b1
    // 2 bits: unused
    // 6 bits: value of b2
    // 2 bits: value of b3
    // 3 bits: unused
    unsigned char b1 : 3, : 2, b2 : 6, b3 : 2;
};
int main()
{
    std::cout << sizeof( S ) << '\n'; // usually prints 2
}
```
The special unnamed bit field of size zero can be forced to break up padding. It specifies that the next bit field begins at the beginning of its allocation unit:
```c
struct S {
    // will usually occupy 2 bytes:
    // 3 bits: value of b1
    // 5 bits: unused
    // 6 bits: value of b2
    // 2 bits: value of b3
    unsigned char b1 : 3;
    unsigned char :0; // start a new byte
    unsigned char b2 : 6;
    unsigned char b3 : 2;
};
int main()
{
    std::cout << sizeof(S) << '\n'; // usually prints 2
/* } */
```
The following properties of bit fields are implementation-defined
- The value that results from assigning or initializing a signed bit field with a value out of range, or from incrementing a signed bit field past its range.
- Everything about the actual allocation details of bit fields within the class object
    - For example, on some platforms, bit fields don't straddle bytes, on others they do
    - Also, on some platforms, bit fields are packed left-to-right, on others right-to-left
:::info
It is implementation-defined whether a plain (neither explicitly signed nor unsigned) char, short, int, long, or long long bit-field is signed or unsigned.
:::
## Friend
### Definition[^2]
The friend declaration appears in a class body and grants a function or another class access to private and protected members of the class where the friend declaration appears.
### Example
1. Designates a function or several functions as friends of this class
    ```cpp
    class Y {
        int data; // private member
        // the non-member function operator<< will have access to Y's private members
        friend std::ostream& operator<<(std::ostream& out, const Y& o);
        friend char* X::foo(int); // members of other classes can be friends too
        friend X::X(char), X::~X(); // constructors and destructors can be friends
    };
    // friend declaration does not declare a member function
    // this operator<< still needs to be defined, as a non-member
    std::ostream& operator<<(std::ostream& out, const Y& y)
    {
        return out << y.data; // can access private member Y::data
    }
    ```
2. (only allowed in non-local class definitions) Defines a non-member function, and makes it a friend of this class at the same time. Such non-member function is always inline.
    ```cpp
    class X {
        int a;
        friend void friend_set(X& p, int i) {
            p.a = i; // this is a non-member function
        }
     public:
        void member_set(int i) {
            a = i; // this is a member function
        }
    };
    ```
### Notes
- Friendship is not transitive (a friend of your friend is not your friend)
- Friendship is not inherited (your friend's children are not your friends)
- When should you use 'friend' in C++?[^3]

## Linkage[^4][^7]
Linkage describes how identifiers can or can not refer to the same entity throughout the whole program or one single translation unit.
:::info
A translation unit is a source file plus all the headers you `#included` in it, from which the compiler creates the object file.
:::

### Internal linkage
- Internal linkage means that storage is created to represent the identifier only for the translation unit being compiled.
- Other translation units may use the same identifier name with internal linkage, or for a global variable, and no conflicts will be found by the linker because separate storage is created for each identifier
- Any of the following names declared at namespace scope have internal linkage
    - variables, functions, or function templates declared static
    - data members of anonymous unions
### External linkage
- External linkage means that a single piece of storage is created to represent the identifier for all translation units being compiled. The storage is created once, and the linker must resolve all other references to that storage.
- Identifiers with external linkage can be accessed from other translation units by declaring them with the keyword extern.

### No linkage
The name can be referred to only from the scope it is in.
Any of the following names declared at block scope have no linkage:
- Variables that aren't explicitly declared extern (regardless of the static modifier)
- Local classes and their member functions
- Other names declared at block scope such as typedefs, enumerations, and enumerators


## Storage class specifiers[^4][^7]
### Auto
The auto specifier was only allowed for objects declared at block scope or in function parameter lists. It indicated automatic storage duration, which is the default for these kinds of declarations. The meaning of this keyword was changed in C++11
### Register
The register storage class is used to define local variables that should be stored in a register instead of RAM (if possible) (for faster access speed). This means that the variable has a maximum size equal to the register size (usually one word) and can’t have the unary & operator applied to it (as it does not have a memory location).

However, there is no guarantee that the variable will be placed in a register or even that the access speed will increase. It is just a hint to the compiler, this keyword was deprecated in C++11.
### Static
- A static global variable or static function at file scope specifies that the variable or function has internal linkage.
- A static variable in a function specifies that the variable retains its state between calls to that function.
    :::info
    Variables declared at block scope with the specifier static have static storage duration but are initialized the first time control passes through their declaration (unless their initialization is zero- or constant-initialization, which can be performed before the block is first entered). On all further calls, the declaration is skipped.
    :::
- A static data member in a class declaration specifies that one copy of the member is shared by all instances of the class.
    - A static data member must be defined at file scope.
    - An integral data type member[^5] that you declare as const static can have an initializer.[^6]
        ```cpp
        #include <iostream>
        using namespace std;
        class A
        {
        public:
            static int num;/* just a declaration */
            static const int num2 = 5;// const static integral type can have initializer
            /* static int num3 = 3;//complier error */
            /* non-const static data member nust be initialized out of line */
        };

        int A::num = 1;
        //The member shall still be defined in file scope

        int main( void )
        {
            cout << A::num << endl;// output 1
            cout << A::num2 << endl;// output 5
            return 0;
        }
        ```
- When you declare a member function in a class declaration, the static keyword specifies that the function is shared by all instances of the class.
    - A static member function cannot access an instance member because the function does not have an implicit this pointer.
    - To access an instance member, declare the function with a parameter that is an instance pointer or reference.
- You cannot declare the members of a union as static. However, a globally declared anonymous union must be explicitly declared static.

### Extern
The extern specifier is only allowed in the declarations of variables and functions (except class members or function parameters). It specifies external linkage, and does not technically affect storage duration, but it cannot be used in a definition of an automatic storage duration object, so all extern objects have static or thread durations. In addition, a variable declaration that uses extern and has no initializer is not a definition.

## Casting[^8][^9][^11]

### Explicit type casting
Explicit type casting is a way of explicitly informing the compiler that you intend to make the conversion and that you are aware that data loss might occur

### C style casting
A C-style cast is defined as the first of the following which succeeds:
- const_cast
- static_cast<new_type>(expression)
- static_cast (with extensions) followed by const_cast;
- reinterpret_cast<new_type>(expression);
- reinterpret_cast followed by const_cast

It can therefore be used as a replacement for other casts in some instances, but can be extremely dangerous because of the ability to devolve into a reinterpret_cast, and the latter should be preferred when explicit casting is needed, unless you are sure static_cast will succeed or reinterpret_cast will fail. Even then, consider the longer, more explicit option.

### static_cast
static_cast can perform conversions between pointers to related classes, not only upcasts (from pointer-to-derived to pointer-to-base), but also downcasts (from pointer-to-base to pointer-to-derived). No checks are performed during runtime to guarantee that the object being converted is in fact a full object of the destination type. Therefore, it is up to the programmer to ensure that the conversion is safe. On the other side, it does not incur the overhead of the type-safety checks of dynamic_cast.

```cpp
#include <vector>
#include <iostream>

struct B {
    int m = 0;
    void hello() const {
        std::cout << "Hello world, this is B!\n";
    }
};
struct D : B {
    void hello() const {
        std::cout << "Hello world, this is D!\n";
    }
};

enum class E { ONE = 1, TWO, THREE };
enum EU { ONE = 1, TWO, THREE };

int main()
{
    // 1: initializing conversion
    int n = static_cast<int>(3.14);
    std::cout << "n = " << n << '\n';
    std::vector<int> v = static_cast<std::vector<int>>(10);
    std::cout << "v.size() = " << v.size() << '\n';

    // 2: static downcast
    D d;
    B& br = d; // upcast via implicit conversion
    br.hello();
    D& another_d = static_cast<D&>(br); // downcast
    another_d.hello();

    // 3: lvalue to xvalue
    std::vector<int> v2 = static_cast<std::vector<int>&&>(v);
    std::cout << "after move, v.size() = " << v.size() << '\n';

    // 4: discarded-value expression
    static_cast<void>(v2.size());

    // 5. inverse of implicit conversion
    void* nv = &n;
    int* ni = static_cast<int*>(nv);
    std::cout << "*ni = " << *ni << '\n';

    // 6. array-to-pointer followed by upcast
    D a[10];
    B* dp = static_cast<B*>(a);

    // 7. scoped enum to int or float
    E e = E::ONE;
    int one = static_cast<int>(e);
    std::cout << one << '\n';

    // 8. int to enum, enum to another enum
    E e2 = static_cast<E>(one);
    EU eu = static_cast<EU>(e2);

    // 9. pointer to member upcast
    int D::*pm = &D::m;
    std::cout << br.*static_cast<int B::*>(pm) << '\n';

    // 10. void* to any type
    void* voidp = &e;
    std::vector<int>* p = static_cast<std::vector<int>*>(voidp);
}
```
### dynamic_cast
dynamic_cast can only be used with pointers and references to classes (or with void*). Its purpose is to ensure that the result of the type conversion points to a valid complete object of the destination pointer type.

This naturally includes pointer upcast (converting from pointer-to-derived to pointer-to-base), in the same way as allowed as an implicit conversion.

But dynamic_cast can also downcast (convert from pointer-to-base to pointer-to-derived) polymorphic classes (those with virtual members) if -and only if- the pointed object is a valid complete object of the target type
```cpp
#include <iostream>
#include <exception>
using namespace std;

class Base { virtual void dummy() {} };
class Derived: public Base { int a; };

int main () {
  try {
    Base * pba = new Derived;
    Base * pbb = new Base;
    Derived * pd;

    pd = dynamic_cast<Derived*>(pba);
    if (pd==0) cout << "Null pointer on first type-cast.\n";

    pd = dynamic_cast<Derived*>(pbb);
    if (pd==0) cout << "Null pointer on second type-cast.\n";

  } catch (exception& e) {cout << "Exception: " << e.what();}
  return 0;
}
```

### reinterpret_cast
reinterpret_cast turns one type directly into another - such as casting the value from one pointer to another, or storing a pointer in an int, or all sorts of other nasty things.It's used primarily for particularly weird conversions and bit manipulations, like turning a raw data stream into actual data, or storing data in the low bits of an aligned pointer.
```cpp
#include<stdio.h>
#include<stdint.h>
int main( void )
{
    int tmp = 1;
    printf("%p\n",&tmp);// print address of tmp
    int64_t addr = reinterpret_cast<int64_t>(&tmp);
    // force complier to cast tmp's address to 64bit integer value
    printf("%lx\n",addr);
    // on my computer(64bit) both print "7fffc56e09dc"
}
```
### const_cast[^10]
used to add/remove const(ness) (or volatile-ness) of a variable.

- Although const cast allows the value of a constant to be changed, doing so is still invalid code that may cause a run-time error. This could occur for example if the constant was located in a section of read-only memory.
- Const cast is instead used mainly when there is a function that takes a non-constant pointer argument, even though it does not modify the pointee.The function can then be passed a constant variable by using a const cast.
```cpp
#include<stdio.h>
void fucn(int *p)
{
    printf("%d",*p);
}
int main( void )
{
    const int myConst=5;
    int *nonConst = const_cast<int*>(&myConst);// remove const attribute
    /* *nonConst = 10; potential run-time error */
    /* fucn(&myConst); error: cannot convert const int* to int* */
    fucn(nonConst);
    return 0;
}
```

## Const[^12][^13]
### basic usage
#### a pointer which points to constant variable
`const int *ptr` or `int const *ptr`
#### a constant pointer which points to variable
`int * const ptr`
#### a constant pointer which points to constant variable
`int const * const ptr`

### const member functions
Declaring a member function with the const keyword specifies that the function is a "read-only" function that does not modify the object for which it is called. A constant member function cannot modify any non-static data members or call any member functions that aren't constant. To declare a constant member function, place the const keyword after the closing parenthesis of the argument list. The const keyword is required in both the declaration and the definition
```cpp
class foo
{
public:
    foo(int x,int y):data(x){}
    void func() const;
private:
    int data;
};
void foo::func() const
{
    this->data= 3;
    // error:can't assign to non-static data member within const member
    cout << data << endl;
}
```
### const return values
```cpp
#include<stdio.h>
const char* func()
{
    return "hello world";
}
int main( void )
{
    func()[0]='a';//could cause write data into read only memory, use const to let complier warn programer
    return 0;
}
```
### parameter passing tip
call by value makes copy of parameter, in C++ we can use call by reference simply pass reference(address) to save the time for copying data, so we can use `const` to prevent data modified.For example `void func(int &a)`.

## 64-bit data models[^14]

| Data model | short (integer) | int | long (integer) | long long | pointers,size_t | Sample operating systems                                                                          |
|------------|-----------------|-----|----------------|-----------|-----------------|---------------------------------------------------------------------------------------------------|
| LLP64      | 16              | 32  | 32             | 64        | 64              | Microsoft Windows (x86-64 and IA-64) using Visual C++; and MinGW                                  |
| LP64       | 16              | 32  | 64             | 64        | 64              | Most Unix and Unix-like systems, e.g., Solaris, Linux, BSD, macOS. Windowswhen using Cygwin; z/OS |
| ILP64      | 16              | 64  | 64             | 64        | 64              | HAL Computer Systems port of Solaris to the SPARC64                                               |
| SILP64     | 64              | 64  | 64             | 64        | 64              | Classic UNICOS[40] (versus UNICOS/mp, etc.)                                                       |

Some experiment
```clike
#include<stdio.h>
#define TYPESIZE(a) printf(#a":%d\n",(int)sizeof(a))
int main( int argc, char *argv[] )
{
    TYPESIZE(char);
    TYPESIZE(short);
    TYPESIZE(int);
    TYPESIZE(long);
    TYPESIZE(long long);
    TYPESIZE(float);
    TYPESIZE(double);
    TYPESIZE(void*);
    return 0;
}
```
```shell
$ gcc -m32 test.c && ./a.out
char:1
short:2
int:4
long:4
long long:8
float:4
double:8
void*:4
$file a.out
a.out: ELF 32-bit LSB executable, Intel 80386
$ gcc -m64 test.c && ./a.out
char:1
short:2
int:4
long:8
long long:8
float:4
double:8
void*:8
$file a.out
a.out: ELF 64-bit LSB executable, x86-64
```
## Alignment[^15]
The type of each member of the structure usually has a default alignment, meaning that it will, unless otherwise requested by the programmer, be aligned on a pre-determined boundary. The following typical alignments are valid for compilers from Microsoft (Visual C++), Borland/CodeGear (C++Builder), Digital Mars (DMC), and GNU (GCC) when compiling for 32-bit x86
* A char (one byte) will be 1-byte aligned.
* A short (two bytes) will be 2-byte aligned.
* An int (four bytes) will be 4-byte aligned.
* A long (four bytes) will be 4-byte aligned.
* A float (four bytes) will be 4-byte aligned.
* A double (eight bytes) will be 8-byte aligned on Windows and 4-byte aligned on Linux (8-byte with -malign-double compile time option).
* A long long (eight bytes) will be 4-byte aligned.
* A long double (ten bytes with C++ Builder and DMC, eight bytes with Visual C++, twelve bytes with GCC) will be 8-byte aligned with C++ Builder, 2-byte aligned with DMC, 8-byte aligned with Visual C++, and 4-byte aligned with GCC.
* Any pointer (four bytes) will be 4-byte aligned. (e.g.: char*, int*)

The only notable differences in alignment for an LP64 64-bit system when compared to a 32-bit system are:
* A long (eight bytes) will be 8-byte aligned.
* A double (eight bytes) will be 8-byte aligned.
* A long long (eight bytes) will be 8-byte aligned.
* A long double (eight bytes with Visual C++, sixteen bytes with GCC) will be 8-byte aligned with Visual C++ and 16-byte aligned with GCC.
* Any pointer (eight bytes) will be 8-byte aligned.

[^1]:[cppreference bit_field](http://en.cppreference.com/w/cpp/language/bit_field)
[^2]:[cppreference friend](http://en.cppreference.com/w/cpp/language/friend)
[^3]:[When should you use ‘friend’ in C++?](http://stackoverflow.com/questions/17434/when-should-you-use-friend-in-c)
[^4]:[Storage class specifiers](http://en.cppreference.com/w/cpp/language/storage_duration)
[^5]:[Fundamental Types (C++)](https://msdn.microsoft.com/en-us/library/cc953fe1.aspx)
[^6]:[static variable in the class declaration or definition?](http://stackoverflow.com/a/11178565)
[^7]:[C++: auto / register / static / const / volatile / linkage / scope](http://yyao.info/c++/2015/03/18/cpp-auto-register-static-const-volatile-linkage-scope)
[^8]:[C++ 的顯性轉型與隱性轉型 - Explicitly/ImplicitlyType Conversion](http://ot-note.logdown.com/posts/173174/note-cpp-named-type-convertion)
[^9]:[When should static_cast, dynamic_cast, const_cast and reinterpret_cast be used?](http://stackoverflow.com/a/332086)
[^10]:[Regular cast vs. static_cast vs. dynamic_cast](http://stackoverflow.com/a/18414172)
[^11]:[Type conversions](http://www.cplusplus.com/doc/tutorial/typecasting/)
[^12]:[C++學習筆記 常數 Const](http://123android.blogspot.tw/2012/02/cconst.html)
[^13]:[The C++ 'const' Declaration: Why & How](http://duramecho.com/ComputerInformation/WhyHowCppConst.html)
[^14]:[64-bit computing](https://en.wikipedia.org/wiki/64-bit_computing)
[^15]:[Data_structure_alignment](https://en.wikipedia.org/wiki/Data_structure_alignment)
