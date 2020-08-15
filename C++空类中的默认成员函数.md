```cpp
class Empty

{

  public:

  Empty(); // 缺省构造函数

  Empty( const Empty& ); // 拷贝构造函数

  ~Empty(); // 析构函数

  Empty& operator=( const Empty& ); // 赋值运算符

  Empty* operator&(); // 取址运算符

  const Empty* operator&() const; // 取址运算符 const

};
```

默认构造函数   
析构函数   
拷贝构造函数   
赋值运算符（operator=）   
取址运算符（operator&）（一对，一个非const的，一个const的）