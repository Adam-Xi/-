# 在C中
+ sizeof是一个单目运算符，strlen是一个库函数
+ sizeof的参数可以是数据的类型，也可以是变量，而strlen只能以结尾为'\0'的字符串作参数
+ 编译器在编译时就计算出了sizeof的结果，而strlen必须在运行时才能计算出来
+ sizeof求的是数据类型所占空间的大小，strlen是求字符串的长度
+ 数组做sizeof的参数不退化，传递给strlen就退化为指针了

```cpp
void TestFunc1()
{
    printf("char = %d\n", sizeof(char));  // 1
    printf("char* = %d\n", sizeof(char*));  // 4

    printf("double = %d\n", sizeof(double));  // 8
    printf("double* = %d\n", sizeof(double*));  // 4
}

void TestFunc2()
{
    char str[20] = "0123456789";
    printf("%d\n", strlen(str));  // 10
    printf("%d\n", sizeof(str));  // 20
}

void TestFunc3()
{
    char *a = "abcdef";
    char b[] = "abcdef";
    char c[] = {'a', 'b', 'c', 'd', 'e', 'f'};

    printf("%d %d\n", sizeof(a), strlen(a));  // 4 6
    printf("%d %d\n", sizeof(b), strlen(b));  // 7 6
    printf("%d %d\n", sizeof(c), strlen(c));  // 6 -(不确定值)
}
/*
当以字符串赋值时，字符串结尾自动加一个'\0'
sizeof(a)：求类型空间大小，32位操作系统下指针类型所指的空间大小为4个字节
strlen(a)：遇到'\0'就结束，求的是字符串的长度，为6个字节

sizeof(b)：求数组所占空间大小，为7个字节
strlen(b)：字符串赋值，求字符串长度，与'\0'就结束，为6个字节

sizeof(c)：数组c以单个元素赋值，没有'\0'结束符，所占空间大小为6个字节
strlen(c)：遇到'\0'结尾字符串的长度，由于没有找到'\0'，所以返回一个不确定的值
*/
```
