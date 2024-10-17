# C++



#### 左值与右值

直观理解：

- 如果能取地址，说明这个变量是左值，可以通过地址修改它
- 如果不能取地址，说明这个变量是右值，不能通过地址修改它

C++11中，左值分为左值，将亡值，右值分为纯右值，将亡值

在c++ 11中，所有的值必须为左值、纯右值、将亡值的三种的任一一种。



#### 左值引用与右值引用

###### 左值引用是具名变量值的别名，右值引用是则是不具名（匿名）变量的别名

右值引用相当于给右值“续期”

所以相比以下声明

```C++
CObj c = RetValue();

CObj && c = RetValue();
```

不使用右值引用就会多一次构造和析构

使用右值引用可以降低临时变量的开销。

```c++
int & func(int)
{
    int b=10;
    return b; //返回的是一个左值引用
}
int main()
{
    int && a=func(10);//此时会提示：“无法将右值引用绑定到左值”
    std::cout<<a<<std::endl;
    return 0;
}
```

但即使将“&&”修改为“&”将a声明为一个左值引用也是不对的。因为b只是一个局部变量当函数执行完成，b随即被销毁。因此会引发错误

```C++
//那么这样返回的值是左值还是右值？
int func()
{
    int b=10;
    return b; 
}
```

在C++中，函数返回局部对象时，无论该对象是左值还是右值，编译器都会隐式地将其转换为右值。这是因为局部对象的生命周期仅在函数作用域内有效，一旦函数返回，这些局部对象的存储空间将被释放。因此，返回的临时对象必须通过移动语义来处理，以避免不必要的拷贝。



#### 万能引用

- 发生类型推导（例如模板、auto）的时候，使用T&&类型表示为万能引用，否则表示右值引用。

- 万能引用类型的形参能匹配任意引用类型的左值、右值。

  这样是合法的：

  ```C++
  template <class T>
  void func(T&& t)
  {
      return;
  }
  int main()
  {
      std::vector<int> a;
      func(std::move(a));
       func(a);
      return 0;
  }
  ```

  但是当func在定义时参数模板为"T& t" 那么func(std::move(a));就是不合法的

  因为std::move()作用是将左值转换为右值。函数指定的参数模板为左值引用。

  

  #### 完美转发 std::forward<T>(t)

  是一个模板函数，它可以在模板函数内给另一个函数传递参数时，将参数类型保持原本状态传入（如果形参推导出是右值引用则作为右值传入，如果是左值引用则作为左值传入。

  ```C++
  template<class F,class ...Args>
      void enqueue(F &&f,Args&&... args)
      {
      	/*
      	希望构造一个function函数对象 task。通过std::bind来绑定函数和参数列表。但是并不知道将要传入的函数与参数是			左值还是右值。因此可以用std::forward来使得它们保持原来的类型.
      	*/
          std::function<void()> task =
              std::bind(std::forward<F>(f),std::forward<Args>(args)...);
          {
              std::unique_lock<std::mutex> lock(mtx);
              tasks.emplace(std::move(task));
          }
          condition.notify_one();
      }
  ```




#### std::function

它允许将可调用对象（函数、函数指针、Lambda表达式等）包装成一个对象，使得可以像操作其他对象一样操作和传递可调用对象.

需要将可调用对象的返回值类型和“需要传入的”参数列表。

一般使用：

```C++
void printSum(int a, int b) {
    std::cout << "Sum: " << a + b << std::endl;
}

int main() {
    std::function<void(int, int)> func = printSum;
    func(3, 4);  // 输出 Sum: 7
    return 0;
}
```

调用类內函数

```C++
class Greeter {
public:
    std::string operator()() const {
        return "hello!";
    }
};
int main() {
    std::function<std::string()> func = Greeter();
    std::string str=func();  *// 调用封装的可调用对象*
    std::cout<<str<<std::endl; //输出hello!
    return 0;
}
```

此处func能够调用Greeter()的前提是他们的函数标签要相同。

那么，如果这个类里没有一个这样的重载()的函数，是否还可以像这样去调用呢。



答案是不能，

```C++
    int func()
    {
        return 10;
    }
```

假定如此：

```C++
    std::function<int()> func = Greeter();
```

这样是不可行的.因为func需要隐式的传入一个“this”，在指定参数列表的时候并没有办法去指定它,因此，二者的函数签名不同，无法进行调用。

那么如何调用类內的一个成员函数呢？

可以使用lamda表达式:

```C++
int main() {
    Greeter greeter;
    //捕获greeter并引用，int a为需要传入的参数,最后返回greeter.func(a)
    std::function<int(int)> func =[&greeter](int a)
    {
       return greeter.func(a);
    };
    std::cout<<func(5)<<std::endl;
    return 0;
}
```

更加灵活的使用：

```C++
   		td::function<void()> task =
            std::bind(std::forward<F>(f),std::forward<Args>(args)...);
```



#### std::bind()

std::bind返回一个基于f的函数对象，其参数被绑定到args上。

上面代码快的代码展示了如何将可变参数列表绑定到函数对象上。

当 `...` 出现在函数参数列表的末尾时，它表示该函数可以接受可变数量的参数。这通常与模板编程结合使用，以创建可以接受任意数量参数的泛型函数或操作符。

还可以使用占位符，使得可以在调用时指定参数：

```C++
void task(int a, int b ,int c)
{
    std::cout<<a<<"+"<<b<<"+"<<c<<"="<<a+b+c<<std::endl;
}
int main() {
    std::function<void(int)> func=std::bind(task,2,3,std::placeholders::_1);
    func(4); //输出2+3+4=9
    return 0;
}
```

占位符数量(1)=std::function<void(int)> func  这里void()括号里的参数列表长度(1),这个数字与bind中已经提供而没有使用占位符的参数数量(2)之和(3)应当等于task函数本身应当具有的参数数量(3)。