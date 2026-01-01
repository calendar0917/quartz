---
creation date: 2025-12-24 11:02
modification date: 2025-12-24 11:02
---
深拷贝类模板：

```cpp
#include <iostream>
#include <cstring>
using namespace std;

class MyString {
private:
    char* data; // 动态内存资源

public:
    // 1. 普通构造函数
    MyString(const char* str = nullptr) {
        if (str == nullptr) {
            data = new char[1];
            *data = '\0';
        } else {
            data = new char[strlen(str) + 1];
            strcpy(data, str);
        }
    }

    // 2. 析构函数 (必背: delete[])
    ~MyString() {
        if (data) {
            delete[] data; // 数组必须用 delete[]
            data = nullptr; //以此防止悬空指针
        }
    }

    // 3. 拷贝构造函数 (Deep Copy)
    // 场景: MyString s2 = s1; 或传值参数
    MyString(const MyString& other) {
        data = new char[strlen(other.data) + 1]; // 申请新资源
        strcpy(data, other.data);                // 复制内容
    }

    // 4. 赋值运算符重载 (最难点: 检查自赋值 + 释放旧资源)
    // 场景: s2 = s1;
    MyString& operator=(const MyString& other) {
        if (this == &other) { // a. 检查自赋值
            return *this;
        }
        delete[] data; // b. 释放旧内存！
        
        data = new char[strlen(other.data) + 1]; // c. 分配新内存
        strcpy(data, other.data);                // d. 复制
        
        return *this; // e. 返回引用支持链式赋值
    }

    // 5. 友元重载 << (输出)
    friend ostream& operator<<(ostream& os, const MyString& s) {
        os << s.data;
        return os;
    }
};
```

多态与虚析构模板

```cpp
class Base {
public:
    Base() { cout << "Base ctor" << endl; }
    
    // !!! 必须是虚析构，否则 delete basePtr 不调子类析构
    virtual ~Base() { cout << "Base dtor" << endl; } 
    
    // 纯虚函数，使 Base 成为抽象类
    virtual void show() = 0; 
};

class Derived : public Base {
public:
    Derived() { cout << "Derived ctor" << endl; }
    ~Derived() { cout << "Derived dtor" << endl; }
    
    // 必须重写纯虚函数，否则 Derived 也是抽象类
    void show() override { 
        cout << "Derived show" << endl; 
    }
};
```
