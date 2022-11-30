### SWIG 简介
`SWIG`是个帮助使用`C`或者`C++`编写的软件能与其它各种高级编程语言进行嵌入联接的开发工具。 `SWIG`能应用于各种不同类型的语言包括常用脚本编译语言例如`Perl`, `PHP`, `Python`, `Tcl`, `Ruby` and `PHP`。
##### 工作流
通过`*.i`文件，声明一些接口`interface`，自动生成一个`*_wrap.c`和一个`*.py`文件。 `*_wrap.c`文件参与C/C++层的编译，一起生成`_*.so`文件，而`*.py`文件则作为上层Python的使用接口。 其价值在于，新增了一个抽象层级，令使用更便捷，而且其中的代码都是自动生成的。
##### example
这里提供一个任意的`example.c++`文件来描述`C++`与`Python`通信
```c++
#include <time.h> 


double My_variable = 3.0;

int fact(int n) {
	if (n <= 1) return 1;
	else return n*fact(n-1);
} 

int my_mod(int x, int y) {
	return (x%y); 
}

char *get_time() {
	time_t ltime;
	time(&ltime);
	return ctime(&ltime);
}
```
以及`example.h`，当然这个文件不是必须的，只是用来展示`interface`文件的一些区别。
```c++
#ifndef EXAMPLE_H
#define EXAMPLE_H

extern double My_variable;
extern int fact(int n);
extern int my_mod(int x,int y);
extern char *get_time();

#endif
```
前面描述的过程简单的讲就是写一个`interface`文件，即可自动生成一个`C/C++`与一个`Python`的包装文件。 让连接`C/C++`与`Python`的这个抽象层级，完全不用手写。

比如，下面的`example.i`文件，暴露了四个接口。

```c++
%module example

%{
#include "example.h"
%}

extern double My_variable;
extern int fact(int n);
extern int my_mod(int x, int y);
extern char *get_time();
```
注意：
- `%module`指定了模块名为`example`。
- `%{}%`则指定了在生成的`example_wrap.c`文件中需要的声明，这里用了一个`example.h`文件。 如果换成下面的四行`extern`，在这个Demo里也是等效的。
- `extern`的四行，声明了需要对外暴露的四个接口。
- 除了`%`部分，其它均遵循`.h`文件的语法。
- 虽然`example.h`的内容和四个`extern`几乎是一致的，而且`%{}%`中也可以直接使用`extern`，但第三部分不能使用`#include`。
##### 编译以及Makefile
笔者一般使用 `swig -python example.i`进行编译
将`example.i`文件转化成`example_wrap.cpp`和 `example.py`文件

这里提供一个`Makefile`示例
```shell
_example.so : example.o example_wrap.o
	g++ -shared example.o example_wrap.o -o _example.so -lpython3.8m 
	
example.o : example.c++
	g++ -c -fPIC -I/usr/include/python3.8m example.c++ 
example_wrap.o : example_wrap.c++ 
	g++ -c -fPIC -I/usr/include/python3.5m example_wrap.c++ example_wrap.c++ example.py : example.i example.h
	swig -python -c++ example.i 
clean: rm -f *.o *.so example_wrap.* example.py* 
test: python3 test.py 
all: _example.so test 

.PHONY: clean test all 

.DEFAULT_GOAL := all
```

上层的Python可以直接调用这四个接口，而无需考虑中间的转换过程。
```python
import example

print('My_varaiable: %s' % example.cvar.My_variable)
print('fact(5): %s' % example.fact(5))
print('my_mod(7,3): %s' % example.my_mod(7,3))
print('get_time(): %s' % example.get_time())
```
##### 通过Setup.py将动态SWIG配置到项目中
`SWIG`的一个优点是，受到了`distutils/setuptools`的原生支持。 在`setup.py`文件中，可以很方便地直接把`*.i`文件作为源文件进行配置，而不用手写编译脚本。
与`ctypes`相比，虽然使用上麻烦一些，但接口更容易使用，也支持`C++`。 最重要的是，可以很方便地让`C/C++`代码在`setup.py`中编译，更容易跨平台。

```python
swig-python-demo 
├── LICENSE 
├── Makefile 
├── setup.cfg 
├── setup.py 
├── src 
│   └── example 
│       ├── example.c 
│       ├── example.h 
│       ├── example.i 
│       ├── __init__.py 
│       ├── _meta.py 
│       ├── stl_example.cpp 
│       ├── stl_example.hpp 
│       └── stl_example.i 
└── tests 
	├── test_example.py 
	└── test_stl_example.py
```

它虽然只有一个`example`模块，但额外包括了`example.i`和`stl_example.i`两个`SWIG`模块。 前者是官方`C`语言的样例，后者是特别提供的`C++`语言`STL`容器样例。

```python
from setuptools import Extension, find_packages, setup 
EXAMPLE_EXT = Extension( 
	name='_example',
	sources=[ 
		'src/example/example.c', 
		'src/example/example.i',
	], 
) 
STD_EXT = Extension(
	name='_stl_example', 
	swig_opts=['-c++'], 
	sources=[ 
		'src/example/stl_example.cpp', 
		'src/example/stl_example.i', 
	],
	include_dirs=[ 
		'src/example',
	],
	extra_compile_args=[ # The g++ (4.8) in Travis needs this 
		'-std=c++11',
	] 
) 
setup(
	  ... 
	  packages=find_packages('src'), 
	  package_dir={'': 'src'}, 
	  ext_modules=[EXAMPLE_EXT, STD_EXT],
	  ...
)
```

`EXAMPLE_EXT`和`STD_EXT`分别是两个`SWIG`的`Extension`。 当然，它们可以合并成一个，这里是为了分别展示C项目与C++项目。

在`setup`函数中，`ext_modules`参数是一个列表（实际上可以是一个Iterable），用来设置这个包所包含的`Extension`。

**注意**：包名都以下划线`_`开头，这是`SWIG`自动生成的模块所约定的。 例如，在`example.i`中定义的模块名是`example`，那么自动生成的Python文件就是`example.py`，而`SWIG`的接口模块则是`_example`，编译后的文件也是`_example.*.so`。
##### 应对生产py模块消失

在使用`SWIG`时，有一个隐藏的问题。 无论是用`bdist`、`bdist_wheel`还是`bdist_rpm`，这类以编译产物打包的命令，都会丢失`SWIG`生成的py文件。 比如这里，会丢失`example.py`和`stl_example.py`两个文件。

之所以说`隐藏`，是因为在这个问题在本地开发时难以发现。 它只有在首次编译打包时，才会出现；如果在不删除这两个生成的py文件的前提下，再次编译、打包，就不会有这个问题。

根本原因在于，在执行`setup`函数进行编译、打包时，`build_ext`是在`build_py`之后执行的。 由于`SWIG`项目的特殊性，这个默认的执行顺序会让自动生成的Python文件在Python部分打包后才生成，所以在程序包里丢失。 再次打包不会遇到问题，也是因为首次编译时已经执行了`build_ext`。

这里提供一个简单的解决方案，让`build_ext`在`build_py`之前执行。
```python
# Build extensions before python modules, 
# or the generated SWIG python files will be missing. 
class BuildPy(build_py): 
	def run(self): 
		self.run_command('build_ext') 
		super(build_py, self).run() 
setup( 
	...
	cmdclass={ 'build_py': BuildPy, }, 
	... 
)
```

实际上，在[Issue 7562: Custom order for the subcommands of build - Python tracker](https://bugs.python.org/issue7562)就描述了这个问题，而以下两个问题也记录了一些解决方案。 如果对此处提供的解决方案不满意，可以进一步参考它们。

-   [c++ - python distutils not include the SWIG generated module - Stack Overflow](https://stackoverflow.com/questions/12491328/python-distutils-not-include-the-swig-generated-module)
-   [python - setup.py: run build_ext before anything else - Stack Overflow](https://stackoverflow.com/questions/29477298/setup-py-run-build-ext-before-anything-else)
### STL 中容器的提供
##### String
在`interface`声明直接通信Python文件中的str，需要在`%module ... `下声明`%include "std_string.i"`
```c++
%module example_string

%include "std_string.i"

%{
 std::string echo(std::string msg);
}%

std::string echo(std::string msg);
```
如果只是一些内容简单、结构复杂的数据交换，可以考虑以某种方式序列化为字符串，到上层再解析为Python层的数据结构。 这里展示的是一个返回自身的`echo`函数，实现如下：
```c++
#include <string>

using namespace std;

string echo(string msg){
	return msg;
}
```
在Python中调用结果如下：

```python
>>> import example_string
>>> msg = example_string.echo('message')
>>> msg
'message'
>>> isinstance(msg, str)
True
```
##### Vector
`vector`需要用`%template`模版类声明一下

```c++
%module example_vector

%include "std_string.i"
%include "std_vector.i"

%{
using namespace std;

vector<string> vector_int2str(vector<int> input);
%}

namespace std {
  %template(StringVector) vector<string>;
  %template(IntVector) vector<int>;
}

using namespace std;

vector<string> vector_int2str(vector<int> input);
```

以上示例，Python层会自动生成`StringVector`和`IntVector`这两个类，作为类型的替代。 这两个类的命名可以随意，它们都实现了`list`的相关协议，可以当作`list`来使用。

示例中的`vector_int2str`函数，就是把`vector`的`int`转换为`string`，实现如下：

```c++
#include <string>
#include <vector>

using namespace std;

vector<string> vector_int2str(vector<int> input) {
    vector<string> result;
    for (vector<int>::const_iterator it = input.begin();
        it != input.end();
        ++it) {
        result.push_back(to_string(*it));
    }
    return result;
}
```
然而，实际在Python层获取返回值时，却是`tuple`类型。 传入时，也可直接使用`list`或`tuple`等类型
```python
>>> import example_vector
>>> data = example_vector.vector_int2str([1, 2, 3])
>>> data
('1', '2', '3')
>>> isinstance(data, tuple)
True
```
##### map
`map`是最常见的的关联容器。 和`vector`一起，可以简单表达一切线性数据结构。 与`vector`类似，使用时也需要用`%template`声明。 此外，它还有一个**特殊情况**。

```c++
%module example_map

%include "std_string.i"
%include "std_map.i"

%{
using namespace std;

map<int, string> reverse_map(map<string, int> input);
%}

namespace std {
  %template(Int2strMap) map<int, string>;
  %template(Str2intMap) map<string, int>;
}

using namespace std;

map<int, string> reverse_map(map<string, int> input);
```

上面的形式，和`vector`类似。 其中，`reverse_map`反转了映射关系，示例代码如下：

```c++
#include <string>
#include <vector>
#include <map>

using namespace std;

map<int, string> reverse_map(map<string, int> input) {
    map<int, string> result;
    for (map<string, int>::const_iterator it = input.begin();
        it != input.end();
        ++it) {
        result[it->second] = it->first;
    }
    return result;
}
```
**特殊情况**就是，虽然`Str2intMap`和`Int2strMap`也实现了`dict`的协议，但是使用时不能直接用Python的`dict`。
```python
str2int = example_map.Str2intMap()
str2int['1'] = 1
str2int['2'] = 2
str2int['3'] = 3
result = example_map.reverse_map(str2int)
assert isinstance(result, example_map.Int2strMap)
for key, value in result.items():
    assert str2int[value] == key
```

### 常用功能
##### 处理输入输出参数
C++包装的一个常见问题是有的C++函数以指针做为函数参数:
```c++
void add(int x, int y, int *result) {
    *result = x + y;
}
```
或
```c++
int sub(int *x, int *y) {
    return *x-*y;
}
```
处理这种情况的最方便方式是使用SWIG库里的`typemaps.i` (关于SWIG库和Typemap见之后内容):
```c++
%module example
%include "typemaps.i"

void add(int, int, int *OUTPUT);
int sub(int *INPUT, int *INPUT);
```

```python
>>> a = add(3,4)
>>> print a
7
>>> b = sub(7,4)
>>> print b
3
```
另一种写法：
``` c++
%module example
%include "typemaps.i"

%apply int *OUTPUT { int *result };
%apply int *INPUT { int *x, int *y};

void add(int x, int y, int *result);
int sub(int *x, int *y);
```
对于既是输入又是输出参数的处理：
```c++
void negate(int *x) {
	*x = -(*x);
}
```


```c++
%include "typemaps.i"
...
void negate(int *INOUT);
```


```python
>>> a = negate(3)
>>> print a
-3
```

对于多个返回参数的处理：
```c++
/* send message, return number of bytes sent, along with success code */
int send_message(char *text, int len, int *success);
```

```c++
%module example
%include "typemaps.i"
%apply int *OUTPUT { int *success };
...
int send_message(char *text, int *success);
```

```python
bytes, success = send_message("Hello World")
if not success:
print "Whoa!"
else:
print "Sent", bytes
```

当输出都通过参数给出情况的处理：
```c++
void get_dimensions(Matrix *m, int *rows, int *columns);

%module example
%include "typemaps.i"
%apply int *OUTPUT { int *rows, int *columns };
```

```c++
void get_dimensions(Matrix *m, int *rows, *columns);
```

```python
>>> r,c = get_dimensions(m)
```
**注意**，typemaps.i只支持了基本数据类型，所以不能写`void foo(Bar *OUTPUT);`，因为typemaps.i里没有对Bar定义OUTPUT规则。

#### C数组实现

有的C函数要求传入一个数组作为参数，调用这种函数时不能直接传入一个Python list或tuple, 有三种方式能解决这个问题：

使用类型映射(Typemap), 将数组代码生成为Python list或tuple相应代码使用辅助函数，用辅助函数生成和操作数组对象，再结合在接口文件中插入一些Python代码，也可使Python直接传入list或tuple。这种方式在之后说明。使用SWIG库里的`carrays.i`  ，这里先介绍carrays.i方式：
```c++
int sumitems(int *first, int nitems) {
    int i, sum = 0;
    for (i = 0; i < nitems; i\+\+) {
        sum += first[i];
    }
    return sum;
}
```

```c++
%include "carrays.i"
%array_class(int, intArray);
```

```python
>>> a = intArray(10000000) # Array of 10-million integers
>>> for i in xrange(10000): # Set some values
... a[i] = i
>>> sumitems(a,10000)
49995000
```
通过 `%array_class` 创建出来的数组是C数组的直接代理，非常底层和高效，但是，它也和C数组一样不安全，一样没有边界检查。

#### C/C++辅助函数

可以通过辅助函数来完一些SWIG本身不支持的功能。事实上，你可以使SWIG支持几乎所有你需要的功能，有很多C++特性是SWIG本身支持或者通过库支持的，不需要通过辅助函数实现。

同样的，直接上例示代码：
```c++
void set_transform(Image *im, double m[4][4]);
```

```python
>>> a = [
... [1,0,0,0],
... [0,1,0,0],
... [0,0,1,0],
... [0,0,0,1]]
>>> set_transform(im,a)
Traceback (most recent call last):
File "<stdin>", line 1, in ?
TypeError: Type error. Expected _p_a_4__double
```
可以看到，set_transform是不能接受Python二维List的，可以用辅助函数帮助实现：
```c++
%inline %{
    /* Note: double[4][4] is equivalent to a pointer to an array double (*)[4] */
    double (*new_mat44())[4] {
    return (double (*)[4]) malloc(16*sizeof(double));
}
void free_mat44(double (*x)[4]) {
    free(x);
}
void mat44_set(double x[4][4], int i, int j, double v) {
    x[i][j] = v;
}
double mat44_get(double x[4][4], int i, int j) {
    return x[i][j];
}
%}
```

```python
>>> a = new_mat44()
>>> mat44_set(a,0,0,1.0)
>>> mat44_set(a,1,1,1.0)
>>> mat44_set(a,2,2,1.0)
...
>>> set_transform(im,a)
```
当然，这样使用起来还不够优雅，但可以工作了，接下来介绍通过插入额外的Python代码来让使用优雅起来。

#### 插入额外的Python代码

为了让set_transform函数接受Python二维list或tuple，我们可以对它的Python代码稍加改造：
```c++
void set_transform(Image *im, double x[4][4]);

...
/* Rewrite the high level interface to set_transform */
%pythoncode %{
    def set_transform(im,x):
    a = new_mat44()
    for i in range(4):
    for j in range(4):
    mat44_set(a,i,j,x[i][j])
    _example.set_transform(im,a)
    free_mat44(a)
%}
```

```python
>>> a = [
... [1,0,0,0],
... [0,1,0,0],
... [0,0,1,0],
... [0,0,0,1]]
>>> set_transform(im,a)
```
SWIG还提供了`%feature("shadow")`,`%feature("pythonprepend")`, `%feature("pythonappend")`来支持重写某函数的代理函数，或在某函数前后插入额外代码，在`%feature("shadow")`中 可用`$action`来指代对C++相应函数的调用：
```c++
%module example

// Rewrite bar() python code

%feature("shadow") Foo::bar(int) %{
def bar(*args):
#do something before
$action
#do something after
%}

class Foo {
    public:
       int bar(int x);
}
```
或者：
```c++
%module example

// Add python code to bar() 

%feature("pythonprepend") Foo::bar(int) %{
#do something before C\+\+ call
%}

%feature("pythonappend") Foo::bar(int) %{
#do something after C\+\+ call
%}

class Foo {
public:
int bar(int x);
}
```
#### 用%extend指示器扩展C++类

你可以通过`%extend`指示器扩展C++类，甚至可用通过这种方式重载Python运算符：
```c++
%module example
%{
#include "someheader.h"
%}

struct Vector {
   double x,y,z;
};

%extend Vector {
char *__str__() {
    static char tmp[1024];
    sprintf(tmp,"Vector(%g,%g,%g)", $self->x,$self->y,$self->z);
    return tmp;
}
Vector(double x, double y, double z) {
    Vector *v = (Vector *) malloc(sizeof(Vector));
    v->x = x;
    v->y = y;
    v->z = z;
    return v;
}

Vector __add__(Vector *other) {
    Vector v;
    v.x = $self->x + other->x;
    v.y = $self->y + other->y;
    v.z = $self->z + other->z;
    return v;
}

};
```

```python
>>> v = example.Vector(2,3,4)
>>> print v
Vector(2,3,4)
>>> v = example.Vector(2,3,4)
>>> w = example.Vector(10,11,12)
>>> print v+w
Vector(12,14,16)
```
**注意**，在`%extend`里`this`用`$self`代替。

#### 字符串处理

SWIG将char_ 映射为Python的字符串，但是Python字符串是不可修改的（immutable），如果某函数有修改char_ ，很可能导致Python解释器崩溃。对由于这种情况，可以使用SWIG库里的`cstring.i`。

#### 模块

`SWIG`通过`%module`指示器指定`Python`模块的名字

#### 函数及回调函数

全局函数被包装为`%module`指示模块下的函数，如：
```c++
%module example
int add(int a, int b);
```

```python
>>>import example
>>>print example.add(3, 4)
7
```

#### 全局变量

`SWIG`创建一个特殊的变量`cvar`来存取全局变量，如：
```c++
%module example
%inline %{
double density = 2.5;
%}
```

```python
>>>import example
>>>print example.cvar.density
2.5
```

inline是另一个常见的SWIG指示器，用来在接口文件中插入C/C++代码，并将代码中声明的内容输出到接口中。

#### 常量和枚举变量

用`#define`, `enum`或者`%constant`指定常量：
```c++
#define PI 3.14159
#define VERSION "1.0"

enum Beverage { ALE, LAGER, STOUT, PILSNER };

%constant int FOO = 42;
%constant const char *path = "/usr/local";
```
#### 指针，引用，值和数组

SWIG完整地支持指针：
```c++
%module example

FILE *fopen(const char *filename, const char *mode);
int fputs(const char *, FILE *);
int fclose(FILE *);
```

```python
>>> import example
>>> f = example.fopen("junk","w")
>>> example.fputs("Hello World\n", f)
>>> example.fclose(f)
>>> print f
<Swig Object at _08a71808_p_FILE>
>>> print str(f)
_c0671108_p_FILE
```
指针的裸值可以通过将指针对象转换成int获得，不过，无法通过一个int值构造出一个指针对象。
```python
>>> print int(f)
135833352
```
`0`或`NULL`被表示为None.

对指针的类型转换或运算必须通过辅助函数完成，特殊要注意的是，对C++指针的类型转换，应该用C++方式的转换，而不是用C方式的转换，因为在转换无法完成是，C++方式的转换会返回NULL，而C方式的转换会返回一个无效的指针：
```c++
%inline %{
/* C-style cast */
Bar *FooToBar(Foo *f) {
    return (Bar *) f;
}

/* C\+\+-style cast */
Foo *BarToFoo(Bar *b) {
   return dynamic_cast<Foo*>(b);
}

Foo *IncrFoo(Foo *f, int i) {
   return f+i;
}
```
在C++中，函数参数可能是指针，引用，常量引用，值，数据等，SWIG将这些类型统一为指针类型处理（通过相应的包装代码）:
```c++
void spam1(Foo *x); // Pass by pointer
void spam2(Foo &x); // Pass by reference
void spam3(const Foo &x);// Pass by const reference
void spam4(Foo x); // Pass by value
void spam5(Foo x[]); // Array of objects
```

```python
>>> f = Foo() # Create a Foo
>>> spam1(f) # Ok. Pointer
>>> spam2(f) # Ok. Reference
>>> spam3(f) # Ok. Const reference
>>> spam4(f) # Ok. Value.
>>> spam5(f) # Ok. Array (1 element)
```
返回值是也同样的：
```c++
Foo *spam6();
Foo &spam7();
Foo spam8();
const Foo &spam9();
```
这些函数都会统一为返回一个Foo指针。

结构和类，以及继承  
结构和类是以Python类来包装的：
```c++
struct Vector {
   double x,y,z;
};
```

```python
>>> v = example.Vector()
>>> v.x = 3.5
>>> v.y = 7.2
>>> print v.x, v.y, v.z
7.8 -4.5 0.0
```
如果类或结构中包含数组，该数组是通过指针来操纵的：
```c++
struct Bar {
	int x[16];
};
```

```python
>>> b = example.Bar()
>>> print b.x
_801861a4_p_int
```

对于数组赋值，SWIG会做数据的值拷贝：
```python
>>> c = example.Bar()
>>> c.x = b.x # Copy contents of b.x to c.x
```
但是，如果一个类或结构中包含另一个类或结构成员，赋值操作完全和指针操作相同。  
对于静态类成员函数，在Python中有三种访问方式:
```c++
class Spam {
    public:
        static int bar;
        static void foo();
};
```

```python
>>> example.Spam_foo() # Spam::foo()
>>> s = example.Spam()
>>> s.foo() # Spam::foo() via an instance
>>> example.Spam.foo() # Spam::foo(). Python-2.2 only
```
其中第三种方式Python2.2及以上版本才支持，因为之前版本的Python不支持静态类成员函数。

静态类成员变量以全局变量方式获取：
```python
>>> print example.cvar.Spam_bar
```
SWIG支持C++继承，可以用Python工具函数验证这一点：
```c++
class Foo {
    ...
};

class Bar : public Foo {
    ...
};
```

```python
>>> b = Bar()
>>> instance(b,Foo)
1
>>> issubclass(Bar,Foo)
1
>>> issubclass(Foo,Bar)
0
```
同时，如果有形如`void spam(Foo *f);`的函数，可以传`b = Bar()`进去。

SWIG支持多继承。

#### 重载

SWIG支持C++重载：
```c++
void foo(int);
void foo(char *c);
```

```python
>>> foo(3) # foo(int)
>>> foo("Hello") # foo(char *c)
```
但是，SWIG不能支持所有形式的C++重载，如：
```c++
void spam(int);
void spam(short);
```
或
```c++
void foo(Bar *b);
void foo(Bar &b);
```
这种形式的声明会让SWIG产生警告，可以通过重名命或忽略其中一个来避免这个警告：
```c++
%rename(spam_short) spam(short);
```
或
```c++
%ignore spam(short);
```
#### 运算符重载

SWIG能够自动处理运算符重载：
```c++
class Complex {
    private:
        double rpart, ipart;
    public:
        Complex(double r = 0, double i = 0) : rpart(r), ipart(i) { }
        Complex(const Complex &c) : rpart(c.rpart), ipart(c.ipart) { }
        Complex &operator=(const Complex &c);
        
        Complex operator+=(const Complex &c) const;
        Complex operator+(const Complex &c) const;
        Complex operator-(const Complex &c) const;
        Complex operator*(const Complex &c) const;
        Complex operator-() const;
        
        double re() const { return rpart; }
        double im() const { return ipart; }
};
```

```python
>>> c = Complex(3,4)
>>> d = Complex(7,8)
>>> e = c + d
>>> e.re()
10.0
>>> e.im()
12.0
>>> c += d
>>> c.re()
10.0
>>> c.im()
12.0
```
如果重载的运算符不是类的一部分，SWIG无法直接支持，如：
```c++
class Complex {
    ...
    friend Complex operator+(double, const Complex &c);
    ...
};
```
这种情况下SWIG是报一个警告，不过还是可以通过一个特殊的函数，来包装这个运算符：
```c++
%rename(Complex_add_dc) operator+(double, const Complex &);
```
不过，有的运算符无法清晰地映射到Python表示，如赋值运算符，像这样的重载会被忽略。

#### 名字空间

名字空间不会映射成Python的模块名，如果不同名字空间有同名实体要暴露到接口中，可以通过重命名指示器解决：
```c++
%rename(Bar_spam) Bar::spam;

namespace Foo {
    int spam();
}

namespace Bar {
    int spam();
}
```
#### 模板

SWIG对C/C++的包装是二进制级别的，但C++模板根本不是二进制级别的概念，所以对模板的包装需要将模板实例化，SWIG提供%template指示器支持这项功能：
```c++
%module example
%{
#include "pair.h"
%}

template<class T1, class T2>
struct pair {
    typedef T1 first_type;
    typedef T2 second_type;
    T1 first;
    T2 second;
    pair();
    pair(const T1&, const T2&);
    ~pair();
};

%template(pairii) pair<int,int>;
```

```python
>>> import example
>>> p = example.pairii(3,4)
>>> p.first
3
>>> p.second
4
```
如果你要同时映射一个模板，以及以这个模板为参数的另一个模板，还要做一点特殊的工作, 比如，同时映射`pair< string, string >`和 `vector< pair <string, string> >`，需要像下面这样做：
```c++
%module testpair
%include "std_string.i"
%include "std_vector.i"
%include "std_pair.i"
%{
#include <string>
#include <utility>
#include <vector>
using namespace std;
%}

%template(StringPair) std::pair<std::string ,std::string>;
SWIG_STD_VECTOR_SPECIALIZE_MINIMUM(StringPair, std::pair< std::string, std::string >);
%template(StringPairVector) std::vector< std::pair<std::string, std::string> >;
```
#### 智能指针

有的函数的返回值是智能指针，为了调用这样的函数，只需要对智能指针类型做相应声明：
```c++
%module example
...
%template(SmartPtrFoo) SmartPtr<Foo>;
...
```

```python
>>> p = example.CreateFoo() # CreatFool()返回一个SmartPtr<Foo>
>>> p.x = 3 # Foo::x
>>> p.bar() # Foo::bar
```
可以通过`p.__deref__()`得到相应的`Foo*`

#### 引用记数对象支持

对于使用引用记数惯例的C++对象，SWIG提供了`%ref`和`%unref`指示器支持，使用Python里使用时不用手工调用`ref`和`unref`函数。因为我们目前没有使用引用记数技术，具体细节这里不详述了。

#### 内存管理

SWIG是通过在Python里创建C++相应类型的代理类型来包装C++的，每个Python代理对象里有一个`.thisown`的标志，这个标志 决定此代理对象是否负责相应C++对象的生命周期：如果`.thisown`这个标志为`1`，Python解释器在回收Python代理对象时也会销毁相应的 C++对象，如果没有这个标志或这个标志的值是`0`，则Python代理对象回收时不影响相应的C++对象。

当创建对象，或通过值返回方式获得对象时，代理对象自动获得`.thisown`标志。当通过指针方式获得对象时，代理对象`.thisown`的值为`0`：
```c++
class Foo {
    public:
        Foo();
        Foo bar();
        Foo *spam();
};
```

```python
>>> f = Foo()
>>> f.thisown
1
>>> g = f.bar()
>>> g.thisown
1
>>> f = Foo()
>>> s = f.spam()
>>> print s.thisown
0
```
当这种行为不是期望的行为的时候，可以人工设置这个标志的值：
```python
>>> v.thisown = 0
```
#### 跨语言多态

当你希望用Python扩展（继承）C++的类型的时候，你就需要跨语言多态支持了。SWIG提供了一个调度者(director)特性支持此功能，但此特性默认是关闭的，通过以下方式打开此特性：

首先，在module指示器里打开
```c++
%module(directors="1") modulename
```
其次，通过%feature指示器告诉SWIG哪些类和函数需要跨语言多态支持：
```c++
// generate directors for all classes that have virtual methods
%feature("director"); 

// generate directors for all virtual methods in class Foo
%feature("director") Foo; 

// generate a director for just Foo::bar()
%feature("director") Foo::bar;
```
可以使用`%feature("nodirector")`指示器关闭某个类型或函数的的跨语言多态支持：
```c++
%feature("director") Foo;
%feature("nodirector") Foo::bar;
```
#### 类型映射(Typemaps)

类型映射是SWIG最核心的一部分，类型映射就是告诉SWIG对某个C类型，生成什么样的代码。不过，SWIG的文档里说类型映射是SWIG的高级自定义部分，不是使用SWIG需要理解的，除非你要提升自己的NB等级

以下的类型映射可用于将整数从Python转换为C:
```c++
%module example

%typemap(in) int {
    $1 = (int) PyLong_AsLong($input);
    printf("Received an integer : %d\n",$1);
}
%inline %{
    int add(int a, int b){
    return a+b;
}
%}
```

```python
>>> import example
>>> example.add(3,4)
Received an integer : 3
Received an integer : 4
7
```
#### SWIG库

SWIG提供了一组库文件，用以支持常用的包装，如数组，标准库等。可以在接口文件中引入这些库文件。比如，在`%include "std_string.i"`后，就可以直接给需要string参数数的函数传Python字符串了。对`"std_vector.i"`举例如下：
```c++
%module example
%include "std_vector.i"

namespace std {
%template(vectori) vector<int>;
};
```

```python
>>> from example import *
>>> v = vectori()
>>> v.push_back(1)
>>> print v.size()
1
```
### 参考资料

-   SWIG和Python: [http://www.swig.org/Doc1.3/SWIGDocumentation.html#Python](https://link.segmentfault.com/?enc=NPawFdpYUeXQXx5iwTyOcA%3D%3D.CfcEnuO72MDCNLI2C4gPeuunJ8KC6inEgOfwPqN1qEjFaW8NWBZJY7eH%2BdsxHEhjqBaXlAo0F9vgeb08XTJMMg%3D%3D)
-   SWIG基础： [http://www.swig.org/Doc1.3/SWIGDocumentation.html#SWIG](https://link.segmentfault.com/?enc=a79sDJFiWjEcEpx5yHzPsQ%3D%3D.KOimyBwyLUn%2F76sQBxqRJUwJDaXRg2wUqs8jGK1NzADGZJy0uBD5WO%2BqrTu0KTm0HdG8e2spv1sCOg38DJXd6g%3D%3D)
-   SWIG和C++: [http://www.swig.org/Doc1.3/SWIGDocumentation.html#SWIGPlus](https://link.segmentfault.com/?enc=kuttZqHMQaVg2ZG7dw6DUg%3D%3D.IV861aqjbMuSuX2x7KIE5jRoL0itaQi25W7l2QpjoVXS%2Fb%2BLwvBk5lVBiel66tLru%2F3ChlBDwF4QGWWKhP%2BUeQ%3D%3D)