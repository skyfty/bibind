boost中有一个bind库， 可以说是一个最为实用的tools了， 但是它与boost结合的有些紧密，而且其中的一些功能并不是很常用，就算将它bcp出独立的库也是一个不小的负担。如果在你的项目中不打算有boost库的痕迹但是又想使用bind的强大功能，那就来看看它吧。


一个一个超小型的bind库， 它实现了大部分boost::bind的功能， 只是将名字空间由boost 变换为 bi 。如果使用了一般的使用中通常可以将

boost::bind(my_fun(), _1,_2)(234, "hello world");

//形式替换为 

bi::bind(my_fun(), _1, _2)(234, "hello world");


既可完成编译，如果使用了名字空间，那就只需要将 using namespace boost 替换为 using namespafce bi 即可完成转化。


同boost::bind一样，它支持任意的函数对象，函数，函数指针，和成员函数指针，它还能将任何参数绑定为一个特定的值，或者将输入的参数发送到任意的位置。


从使用者方向看只是将bind 从boost名字空间移动到了 bi名字空间， 如果使用了 using namespace boost;的方式引入的boost::bind, 那么你很幸运！只要改成using namespace bi;就完成了。


同函数及函数指针一起使用 bind


给定一些函数定义：

int f(int a, int b)
{
  return a + b;
}

int g(int a, int b, int c)
{
  return a + b + c;
}
bind(f, 1, 2) 会产生一个函数对象，它将f(1, 2)这样的函数调用包装到了自身，同样，bind(g, 1, 2, 3)() 的形式调用等价于 g(1, 2, 3)。


而且只绑定一部分参数也是有可能的。bind(f, 1, 5)(x) 等价于 f(x, 5)，这里，_1 是一个占位符参数，它的含义是“用第一个输入参数取代”。


这个实现支持最多 9 个参数的函数对象，同时也只提供了最大一9的占位符 _9。这只是一个实现细节，不是设计的固有限制，从理论上说如果有需要可以提供任意多个参数的函数调用。


bind 能够处理带有两个以上参数的函数，而且它的参数取代机制更为直观：
bind(f, _2, _1)(x, y); // f(y, x)

bind(g, _1, 9, _1)(x); // g(x, 9, x)

bind(g, _3, _3, _3)(x, y, z); // g(z, z, z)

bind(g, _1, _1, _1)(x, y, z); // g(x, x, x)
注意，最后一个示例中，由 bind(g, _1, _1, _1) 生成的函数对象不包含对第一个参数以外的任何参数的引用，但是它仍然能使用一个以上的参数。所有多余的参数被悄悄地忽略，就像在第三个示例中，第一和第二个参数被忽略。


bind 返回的函数对像同标准库其它组件一样具有值语义，所以它持有输入参数，并内部持有函数对象拷贝。例如，在下面的代码中：
int i = 5;

bind(f, i, _1);
一个 i 的值的拷贝被存储于bind生成的函数对象中。


在表达式 bind(f, a1, a2, ..., aN) 中，函数对象 f 必须能够持有正好 N 个参数。这个错误通常在“绑定时间”被查出。换句话说，这个编译错误会被报告在 bind() 被调用的那一行：
int f(int, int);

int main()
{

  bi::bind(f, 1); // error, f函数f必须具备两个参数

  bi::bind(f, 1, 2); // OK
}
这个错误的一个常见变化是忘记成员函数有一个隐含的 "this" 参数：


struct X
{

  int f(int);
};

int main()
{

  bi::bind(&X::f, 1); // error,函数f必须具备两个参数

  bi::bind(&X::f, _1, 1); // OK
}
和函数对象一起使用 bind


bind 并不限于函数，它可以接受任何函数对象。通常情况下，生成的函数对象的 operator() 的返回类型必须显式指定（没有 typeof 操作符，返回类型无法推导）：
struct F
{

  int operator()(int a, int b) { return a - b; }

  bool operator()(long a, long b) { return a == b; }
};

F f;

int x = 104;

bi::bind(bi::type<int>(), f, _1, _1)(x);
bi::bind库不支持 bind<int>(f, _1, _1)(x); 形式的调用， boost支持这种调用方式。


当函数对象暴露了一个名为 result_type 的内嵌类型时，显式返回类型可以被省略：
int x = 8;

bind(std::less<int>(), _1, 9)(x); // x < 9
bind(f, ...) 和 bind<R>(f, ...) 有什么不同


第一个形式指示 bind 去检查 f 的类型以确定它的 arity（参数数量）和返回类型。参数数量错误将在“绑定时间”查明。当然，这样的语法对 f 有一定的要求。它必须是一个函数，函数指针，成员函数指针，或定义了一个内嵌的名为 result_type 的类型的函数对象，简而言之，它必须是某种 bind 可以识别的东西。


第二个形式指示 bind 不要试图识别 f 的类型。它通常和那些没有或不能暴露 result_type 的函数对象一起使用，但是它也能和非标准函数一起使用。例如，当前实现不能自动识别像 printf 这样的可变参数函数，所以你必须用 bind<int>(printf, ...)。注意，有一种可选的 bind(type<R>(), f, ...) 语法因为可移植性的原因也被支持。


另一个需要考虑的重要因素是：当 f 是一个函数对象时，不支持模板偏特化或函数模板部分排序的编译器不能处理第一种形式，而且，大部分情况下，当 f 是一个函数（指针）或成员函数指针时，不能处理第二种形式。


和成员指针一起使用 bind


成员函数的指针和数据成员的指针不是函数对象，因为它们不支持 operator()。为了方便起见，bind 接受成员指针作为它的第一个参数，而它的行为就像使用 std::mem_fn 将成员指针转换成一个函数对象一样。同时bind自然支持boost::shared_ptr 对像的指针，而且不需要应引boost库。


bind(&X::f, _args)


示例：
struct X
{

  bool f(int a);
};

X x;

Boost::shared_ptr<X> p(new X); //自然支持 boost::shared_ptr

int i = 5;

bind(&X::f, x, 1)(i); // ( _internal copy of x).f(i)

bind(&X::f, p, 1)(i); // ( _internal copy of p)->f(i)
最后两个示例的有趣之处在于它们生成“自包含”的函数对象。bind(&X::f, x, _1) 存储 x 的一个拷贝。bind(&X::f, p, _1) 存储 p 的一个拷贝，而且因为 p 是一个 boost::shared_ptr，这个函数对象保存一个属于它自己的 X 的实例的引用，而且当 p 离开它的作用域或者被 reset() 之后，这个引用依然保持有效。


绑定一个被重载的函数的企图通常对导致一个错误，因为无法表示到底要绑定哪一个重载版本。对于带有 const 和非 const 两个重载的成员函数来说，这是一个很常见的问题，就像这个简化的示例：
struct X

{

int& get();

  int const& get() const;

};

int main()

{

  bi::bind( &X::get, _1 );

}
这里的二义性可以通过将（成员）函数指针强制转换到想要的类型来解决：


int main()
{

  bi::bind( static_cast< int const& (X::*) () const >( &X::get ), _1 );

}
另一个或许更可读的办法是引入一个临时变量：


int main()
{

  int const& (X::*get) () const = &X::get;

  bi::bind( get, _1 );

}
为函数组合使用嵌套的 binds


传给 bind 的某些参数可以嵌套 bind 表达式自身：
bind(f, bind(g, _1))(x); // f(g(x))
当函数对象被调用的时候，如果没有指定顺序，内部 bind 表达式先于外部 bind 表达式被求值，在外部 bind 表达式被求值的时候，用内部表达式的求值结果取代它们的占位符的位置。在上面的示例中，当用参数列表 (x) 调用那个函数对象的时候，bind(g, _1)(x) 首先被求值，生成 g(x)，然后 bind(f, g(x))(x) 被求值，生成最终结果 f(g(x))。


bind 的这个特性可被用来执行函数组合。


注意第一个参数——被绑定函数对象——是不被求值的，即使它是一个由 bind 生成的函数对象或一个占位符参数，所以下面的示例不会如你所愿地工作：


typedef void (pf)(int);

std::vector<pf> v;

std::for_each(v.begin(), v.end(), bind(_1, 5));
尽管在缺省情况下，第一个参数是不被求值的，而所有其它参数被求值。但有时候不需要对第一个参数之后的其它参数求值，甚至当它们是内嵌 *bind 子表达式的时候也不需要。

示例


和标准算法一起使用 bind

class image;

class animation

{

public:

   void advance(int ms);

   bool inactive() const;

   void render(image & target) const;

};

std::vector<animation> anims;

template<class C, class P> void erase_if(C & c, P pred)

{

   c.erase(std::remove_if(c.begin(), c.end(), pred), c.end());

}

void update(int ms)

{

   std::for_each(anims.begin(), anims.end(),     ni::bind(&animation::advance, _1, ms));

  erase_if(anims, bi::mem_fn(&animation::inactive));

}

void render(image & target)

{

  std::for_each(anims.begin(), anims.end(), bi::bind(&animation::render, _1, bi::ref(target)));

}
局限性


作为一个通用规则，由 bind 生成的函数对象以引用方式持有它们的参数，因此，不能接受非 const 临时对象或字面常量。


这个库以下面这种形式的识别标识


template<class T> void f(T & t);


接受任意类型的参数并将它们不加改变地传递。注意，这不能用于非 const 右值。


在支持函数模板部分排序的编译器上，一个可能的解决方案是增加一个重载：


template<class T> void f(T & t);


template<class T> void f(T const & t);


很不幸，对于 9 个参数，这样需要提供 512 个重载，这是不切实际的，所以同boost::bind一样，它只选择实现了一个小的子集：对于不大于两个参数的情况，完整地提供了常量重载，对于三个及更多参数，它为所有参数都以常引用方式持有的情况提供了单一的补充重载。这覆盖了使用情况的一个合理的部分。




bi::bind没有打算支持boost库， 如果在你的项目中使用了其它的boost‘tools，那你就不需要它了（或许也可以拿来研究研究），既然使用了boost,就不在乎在使用boost::bind了，所以boost::bind中涉及到与其它boost库的有交集的功能bi::bind库都没有支持。


它自然支持 boost::shared_ptr， 同时对c++10 保标准的shared_ptr也有很好的支持。


bi::bind借鉴了boost::bind的一些思想，有些代码甚至是直接从bind.hpp中复制过来的，但它在牺牲一些扩展性和多平台支持性的后果下， 同样的代码比boost::bind中有40%的效率提升 。


bi-bind 同时提供了 callback功能， 它就像是一个简化版本的 boost::function， 对一次调用行为做了抽象。


int fun1(int p2)
{
   std::cout << "call fun1" << std::endl;
   return p2 *  p2;
}

struct fun2
{
   fun2(int p1)
   {
      p_ = p1;
   }

   int operator()(int p2) const
   {
      return p2 * p_;
   }

   int p_;
};
//绑定一般函数 构造 callback
bi::callback<int(int)> cb = bi::bind(&fun1, _1);
int r = cb(2);   //r=4,使用了实际参数2


//绑定仿函数对象赋值给callback
cb = bi::bind(fun2(2), 9);
r = cb(100);     //r=18,使用了绑定参数9
bind 通常都接收它们的参数的拷贝, 并会在bind_t对像中保存一份相同的对象，而有些对像不具备拷贝语义或是拷贝的代价很大。 为了解决这个问题对，bind对像的可以使用ref 方式将这个对像封装起来使用。它定义了类模板 reference_wrapper<T>和 两个返回实例的函数 ref 和 cref。


reference_wrapper<T> 的目的是容纳一个引向类型为 T 的对象的引用。它主要用于把引用传给bind_t对像， bint_t对像中有对此的重载，即可以解开引用，调用实际的对像。


struct fun2
{
   fun2(int p1)
   {
      p_ = p1;
   }

   int operator()(int p2) const
   {
      return p2 * p_;
   }

   int p_;
};
//假设fun2的拷贝复制的负担很大
fun2 f(23);


//使用bi::ref解除对f对像的值语义调用
bi::bind(bi::ref(f), 9)(2);