#### 1.C++的智能指针

智能指针实际上是一个栈对象，并非是指针类型，在栈对象生命周期即将结束时，智能指针通过析构函数释放由它管理的堆内存。

##### 1.1不适用智能指针会造成的问题

- 忘记delete内存。忘记释放动态内存常会导致“内存泄漏”问题，因为这种内存永远不可能归还给自由空间了。

- 使用已经释放掉的对象。通过在释放内存后将指针置为空，有时可以检测出这种错误。如果不置为空，那么会产生空悬指针（野指针）。这会造成灾难性的后果。

- 同一块内存释放两次，当有两个指针指向相同的动态分配对象时，可能发生这种错误。如果对其中一个指针进行了delete操作，对象的内存就被归还给自由空间了，如果随后又对第二个指针进行delete操作，自由空间就可能被破坏。

##### 1.2如何检测内存泄漏

###### windows下的内存泄漏检测

```C++
#define CRTDBG_MAP_ALLOC  //放在程序最前
#include <iostream>
#include <stdlib.h>    
#include <crtdbg.h> 
using namespace std;

int main()
{
	int *a = new int [10];
	int *p = new int[1000];
	_CrtDumpMemoryLeaks();  //放在程序最后
	system("pause");
	return 0;
}
```

###### Linux 下的内存泄漏检测

```c++
编译：g++ -g -o test test.cpp
使用：valgrind --tool=memcheck ./test
```

##### 1.3为什么C++11会放弃auto_ptr？

我们来看一段代码

```c++
void TestAutoPtr2() 
{
  std::auto_ptr<Simple> my_memory(new Simple(1));
  if (my_memory.get()) 
  {
    std::auto_ptr<Simple> my_memory2;   // 创建一个新的 my_memory2 对象
    my_memory2 = my_memory;             // 复制旧的 my_memory 给 my_memory2
    my_memory2->PrintSomething();       // 输出信息，复制成功
    my_memory->PrintSomething();        // 崩溃
  }
}
```

> 在上述的代码运行时，导致崩溃的原因是执行my_memory2 = my_memory后，my_memory2 取得了my_memory的内存管理权，导致my_memory悬空。

##### 1.4 shared_ptr

shared_ptr可以用来实现共享所有权的概念。多个智能指针可以引用同一个对象，当最后一个智能指针销毁时，对象销毁。每个shared_ptr对象在内部指向两个内存位置（1.指向对象的指针。2.用于控制引用计数的指针）

```c++
1.	#include <iostream>  
2.	#include  <memory> // 需要包含这个头文件  
3.	  
4.	int main()  
5.	{  
6.	    // 使用 make_shared 创建空对象  
7.	    std::shared_ptr<int> p1 = std::make_shared<int>();  
8.	    *p1 = 78;  
9.	    std::cout << "p1 = " << *p1 << std::endl; // 输出78  
10.	  
11.	    // 打印引用个数：1  
12.	    std::cout << "p1 Reference count = " << p1.use_count() << std::endl;  
13.	  
14.	    // 第2个 shared_ptr 对象指向同一个指针  
15.	    std::shared_ptr<int> p2(p1);  
16.	  
17.	    // 下面两个输出都是：2  
18.	    std::cout << "p2 Reference count = " << p2.use_count() << std::endl;  
19.	    std::cout << "p1 Reference count = " << p1.use_count() << std::endl;  
20.	  
21.	    // 比较智能指针，p1 等于 p2  
22.	    if (p1 == p2) {  
23.	        std::cout << "p1 and p2 are pointing to same pointer\n";  
24.	    }  
25.	  
26.	    std::cout<<"Reset p1 "<<std::endl;  
27.	  
28.	    // 无参数调用reset，无关联指针，引用个数为0  
29.     p1.reset();  
30.	    std::cout << "p1 Reference Count = " << p1.use_count() << std::endl;  
31.	      
32.	    // 带参数调用reset，引用个数为1  
33.	    p1.reset(new int(11));  
34.	    std::cout << "p1  Reference Count = " << p1.use_count() << std::endl;  
35.	  
36.	    // 把对象重置为NULL，引用计数为0  
37.	    p1 = nullptr;  
38.	    std::cout << "p1  Reference Count = " << p1.use_count() << std::endl;  
39.	    if (!p1) {  
40.	        std::cout << "p1 is NULL" << std::endl; // 输出  
41.	    }  
42.	    return 0;  
43.	}  


```

##### shared_ptr 的实现

为了实现智能指针的效果必须借助一个计数器，以便随时获知有多少智能指针绑定在同一个对象上，但是这个智能指针不能是smartpointer的一部分，这是因为如果计数器是智能指针的一部分那么每次增加减少计数器的值都必须广播到每一个管理着目标对象的智能指针。这样的代价很大。

以下是智能指针的实现。

```c++
#include <memory>
#include <iostream>

using namespace std;
template <typename T>
class SmartPtr
{
public:
	SmartPtr(T * ptr = nullptr):m_Ptr(ptr)
	{
		if (m_Ptr) 
		{
			m_ReferenceCount = new int(1);
		}
		else
		{
			m_ReferenceCount = new int(0);
		}
	}
	~SmartPtr() 
	{
		(*this->m_ReferenceCount)--;
		if (this->m_ReferenceCount == 0) 
		{
			delete this->m_Ptr;
			delete this->m_ReferenceCount;
		}
	}

	int use_count() 
	{
		return *this->m_ReferenceCount;
	}

	SmartPtr(const SmartPtr& ptr) 
	{
		if (this != &ptr) 
		{
			this->m_Ptr = ptr.m_Ptr;
			this->m_ReferenceCount = ptr.m_ReferenceCount;
			(*this->m_ReferenceCount)++;
		}
	}

	SmartPtr operator=(const SmartPtr& ptr) 
	{
		if (this->m_Ptr == ptr.m_Ptr) 
		{
			return *this;
		}
		if (this->m_Ptr) 
		{
			(*this->m_ReferenceCount)--;
			if (this->m_ReferenceCount == 0) 
			{
				delete this->m_Ptr;
				delete this->m_ReferenceCount;
			}
		}
		this->m_Ptr = ptr.m_Ptr;
		this->m_ReferenceCount = ptr.m_ReferenceCount;
		(*this->m_ReferenceCount)++;
		return *this;
	}

	T& operator*() 
	{
		if (this->m_Ptr)
		{
			return *(this->m_Ptr);
		}
	}

	T& operator->() 
	{
		if (this->m_Ptr) 
		{
			return this->m_Ptr;
		}
	}
private:
	T* m_Ptr;
	int *m_ReferenceCount;
};

```

