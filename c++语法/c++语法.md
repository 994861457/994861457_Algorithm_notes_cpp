# 1.内存
## 1.1 c++语言的内存四区https://blog.csdn.net/qq_38769551/article/details/103099014  
类（未实例化），成员函数，函数，静态函数保存在代码区
## 1.2 操作系统的内存管理与c++。https://blog.csdn.net/quicmous/article/details/98755334
## 1.3 c++内存对齐https://zhuanlan.zhihu.com/p/30007037  
sizeof(struct)的计算
## 1.4 c++各数据类型的内存大小（一个字节8位）https://blog.csdn.net/zcyzsy/article/details/77935651
## 1.5 c++函数调用栈https://www.cnblogs.com/zuixime0515/p/13663796.html    
函数的栈为ebp（底） ~ esp（顶）这俩寄存器保存的地址之间。函数调用时，ebp入栈保存，然后esp内地址保存到。函数调用时参数入栈是从右往左，为了能够实现动态参数。
## 1.6 虚函数表https://zhuanlan.zhihu.com/p/75172640
## 1.7 malloc/free/new/delete  

# 2.面向对象   
## 2.1 多态：多种形态，具体点就是去完成某个行为，当不同的对象去完成时会产生出不同的状态。  
静态多态：模板，函数重载 ：在编译阶段完成，编译器可以优化，效率高  
动态多态：虚函数 : 通过虚函数在运行期实现。处理同一继承体系下异质对象集合的强大威力
## 2.2 虚函数https://www.cnblogs.com/weiyouqing/p/7544988.html  
各个子类重写这些虚函数，以完成具体的功能。客户端的代码（操作函数）通过指向基类的引用或指针来操作这些对象，对虚函数的调用会自动绑定到实际提供的子类对象上去。
# 3.c++11新特性：
# n.待分类  
## n.1 函数指针：只写函数名默认当做函数指针，函数指针解引用加括号然后再用括号写入参数就可以调用。  
