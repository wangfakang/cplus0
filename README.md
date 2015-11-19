一：STL之空间配置器总结    
===

内存池的总结：      
  首先要看用户需要的空间是否在128字节之内，若大于128，则直接使用一级空间配置器，直接进行内存分配，否则使用内存池：
具体过程是：       

```   
1： 首先根据用户需求大小，把其调节成8的倍数，然后在16个自由空闲链表中去查找  
(((bytes) + __ALIGN - 1) & ~(__ALIGN-1));   
2：若找到则把那块从Free_list中分配给用户    
((bytes + __ALIGN -1) / __ALIGN -1);    
3：若恰好找到的那一块是NULL的，则调用refill(ROUND_UP(n))函数，refill函数的意思就是重新填充free_list链表，
在STL中规定的是为ree_list为NULL的那块在内存池中选取20块对应大小的内存分配给他，调用chunk_alloc(n,nobjs)函数，
当内存池无法满足分配20块的时候则最少就要选取1块分配给他

4：倘若此时内存池中剩下的内存大小比用户所需内存类型的1块还小的时候，则重新开辟池子（使用malloc其大小为：
用户所需大小的2倍加上池子以前总大小的十六分之一即：size_t types_to_get   =2*total_types + ROUND_UP(heap_size>>4);）

5：若此时开辟失败则在free_list中为NULL的下一块开始选择一块不为NULL的分配给内存池，然后若是成功则递归调用
chunk_alloc(n,nobjs)函数

6：若此时还是不可以满足的话则调用一级空间配置器来分配（allocate(types_to_get)函数首先还是会直接调用malloc函数，
然后当失败的时候就会调用oo_malloc函数，oo_malloc函数首先会检查my_malloc_handler函数指针是否是NULL，若NULL的话
就直接抛出异常，否则调用my_malloc_handler函数（意思是看用户可不可以还一些内存给系统），然后再重新调用malloc函数）

[是free_list中所存储的obj*]
```



二：STL之iterator类型的萃取技术总结：

首先迭代器的继承层次：       

   输入迭代器              输出迭代器        


               正向迭代器

               双向迭代器
                 
               随机迭代器
              

输入迭代器：所指之物不允许外界改变，只可以读只可以operator++     
输出迭代器：只可以用于写  只可以operator++   
正向迭代器：可以做读写操作  只可以operator++   
双向迭代器：可以双向做读写操作  即可以重载operator++   operator—   
随机迭代器：由于操作的内存是连续的所以不仅可以重载operator++   operator—    
还可以p+n等   

二：迭代器的类型的萃取技术   
====

* 1：为何产生此技术：    
     其迭代器的类型的萃取技术主要用于一些算法里面比如insert当中以及copy，fill当中，
     主要是根据迭代器所指之物的类型的构造函数是否是无关紧要的，如果是无关紧要则直接
     进行移动（即有空间即可操作），否则要通过调用构造函数一个一个的进行构造，所以通
     过迭代器的类型的萃取技术从而可以提高效率  

（const_iterator 与iterator的区别：只是对所指之物的限定）
``` c
const_reference operator*() const    
{   
return _Acc::_Value(_Ptr);   
}
reference operator*() const    
{  
return _Acc::_Value(_Ptr);  
}   
 ```
* 2：此技术是如何做到的

   比方说是List中的迭代器，他是从Bidit双向迭代器继承过来的，而Bidit双向迭代器又是从
   iterator鼻祖继承过来的，当List中迭代器实参推演的时候就会一层一层的向上推演，直到
   鼻祖为止，而鼻祖iterator中把五中类型都进行了封装，然后通过iterator_traits得到迭代
   器的类型，以及所指之物的类型  
   
``` c
template<class Category,class T,class Distance = ptrdiff_t,
         class Pointer = T*,class Reference = T&>
struct iterator//是最传统的迭代器[只是所有迭代器的鼻祖]
{
    typedef Category  iterator_category;//指的是迭代器的类型
    typedef T         value_type;//迭代器所指之物的类型
	typedef Distance  difference_type;//差值类型
	typedef Pointer   pointer;//指针类型
	typedef Reference reference;//引用类型
};


template<class _Ty, class _D>
struct _Bidit [主要是用于List中]: public iterator<bidirectional_iterator_tag,
		                       _Ty, _D> 
{};

template<class _Ty, class _D>
struct _Ranit : public iterator<random_access_iterator_tag,
		                        _Ty, _D> 
{};
template<class Iterator>
struct iterator_traits
{[此步是真正的类型萃取]
	//iterator_traits()
	//{
	//	std::cout<<"iterator<iteator>"<<std::endl;
	//}
	typedef typename Iterator::iterator_category iterator_category;
	typedef typename Iterator::value_type        value_type;
	typedef typename Iterator::difference_type   difference_type;
	typedef typename Iterator::pointer           pointer;
	typedef typename Iterator::reference         reference;
};

template<class T>
struct iterator_traits<T*>
{
	//iterator_traits()
	//{
	//	std::cout<<"iterator<T*>"<<std::endl;
	//}
	typedef random_access_iterator_tag  iterator_category;
	typedef T                           value_type;
	typedef ptrdiff_t                   difference_type;
	typedef T*                          pointer;
	typedef T&                          reference;
};
    

template<class T>
struct iterator_traits<const T*>
{
	typedef random_access_iterator_tag  iterator_category;
	typedef T                           value_type;
	typedef ptrdiff_t                   difference_type;
	typedef const T*                    pointer;
	typedef const T&                    reference;
};

template<class Iterator>
inline typename iterator_traits<Iterator>::iterator_category
iterator_category(const Iterator &)
{
	//iterator_traits<Iterator>();
	typedef typename iterator_traits<Iterator>::iterator_category category;
	return category();
}
template<class Iterator>
inline typename iterator_traits<Iterator>::difference_type *
distance_type(const Iterator &)
{
	return static_cast<typename iterator_traits<Iterator>::difference_type *>(0);
}

template<class Iterator>
inline typename iterator_traits<Iterator>::value_type *[之所以要用指针是由于我们可以明确指针的大小4字节，
//如果返回对象的话可能会大于4字节，所以在此也考虑到了效率]
value_type(const Iterator &)
{
	return static_cast<typename iterator_traits<Iterator>::value_type*>(0);
}

template<class InputIterator>
inline typename iterator_traits<InputIterator>::difference_type
__distance(InputIterator first,InputIterator last, 
		                       input_iterator_tag)
{
	typename iterator_traits<InputIterator>::difference_type n = 0;
	for(; first != last; ++first)
	{
		++n;
	}
	return n;
}


template<class RandomAccessIterator>
inline typename iterator_traits<RandomAccessIterator>::difference_type
__distance(RandomAccessIterator  first,RandomAccessIterator last, 
		                                     random_access_iterator_tag )
{
	return last-first;
}

template<class InputIterator>
inline typename iterator_traits<InputIterator>::difference_type
distance(InputIterator first,InputIterator last)
{
	typedef typename iterator_traits<InputIterator>::iterator_category category;
	return __distance(first,last,category());
}


///////////////////////////////////////////////
template<class InputIterator,class Distance>
inline void __advance(InputIterator &i,Distance n,
					  input_iterator_tag)
{
	while(n--) ++i;
}

template<class BidirectionalIterator,class Distance>
inline void __advance(BidirectionalIterator &i,Distance n,
					  bidirectional_iterator_tag)
{
	if(n>=0)
		while(n--) ++i;
	else
		while(n++) --i;
}

template<class RandomAccessIterator,class Distance>
inline void __advance(RandomAccessIterator &i,Distance n,
					          random_access_iterator_tag)
{
	i+=n;
}

template<class InputIterator,class Distance>
inline void advance(InputIterator &i,Distance n)
{
	__advance(i,n,iterator_category(i));
}
```
* 3：此技术的运用     
   比方上面求差值函数distance(first，last)中调用的__distance(first,last,category())函数
就是通过第三个参数来萃取出迭代器的类型，然后选择相应的函数，若是随机迭代器则_diss=last=fast;
否则的话就要通过循环来累加了   


三：STL之Find算法
====
首先是SGI版本的：      
 
 ``` c   
Template<class iterator ,class T>
Iterator   find(Iterator  first , Iterator  last,const T &x)
{
	While(first!=last &&*first!=x)
{
   ++first;
}
return first;
}
PG版本的
template<class _II, class _Ty> inline
_II find(_II _F, _II _L, const _Ty& _V)
{
for (; _F != _L; ++_F)
		if (*_F == _V)
			break;
	return (_F); 
}
```


四：STL之search算法
====
注意：SGI的版本是比PG的版本高的  
  SGI的思想就是：    
1：首先计算出两个要比较的迭代器之间的差值记作为d1与d2   
2：然后通过比较d1与d2的大小，若d1<d2 则直接返回last1    
3：否则定义curent1=fast1   curent2=fast2 ，然后遍历第二个区间（即【fast2，last2】使用while（curent2！=last2）））    
4：在while中若*current1==*current2  则都向后面跑   
5：否则判断d1==d2是否满足，若满足则返回last1  
6：否则重置curent2=fast2；curent1=++fast1 继续while循环    
7：若退出循环则说明在【fast1，last1】中找到了与【fast2，last2】完全匹配的 返回fast1    

* 首先是SGI版本的：  

``` c
template<class _FI1, class _FI2> inline
	_FI1 search(_FI1 _F1, _FI1 _L1, _FI2 _F2, _FI2 _L2)//[意思是看F1到L1之间是否存在F2之L2完全匹配的子序列，若存在则返回F1否则返回L1]
	{return (_Search(_F1, _L1, _F2, _L2,
		_Dist_type(_F1), _Dist_type(_F2))); }
template<class _FI1, class _FI2, class _Pd1, class _Pd2> inline
	_FI1 _Search(_FI1 _F1, _FI1 _L1, _FI2 _F2 , _FI2 _L2,
		_Pd1 *, _Pd2 *)
	{_Pd1 d1 = 0;
	_Distance(_F1, _L1, _D1);
	_Pd2 d2 = 0;
	_Distance(_F2, _L2, _D2);
  ///////////////////////////////////////////
  If(d1<d2)return _L1;

  _FI1 curent1=_F1;
  _F2 curent2=_F2;
 
  While(curent2!=L2)
{
If(*curent1==*curent2)
{
  ++curent1;
++curent2;
}
Else
{
  If(d1==d2) return L1;
  Curent1=++F1;
  Curent2=F2;
  --d1;
}

}
  Return F1;
  
}
```

* PG版本：
PG的思想：其本质是和SGI是一样的，只不过是使用的是for循环，并且PG版本是在循环中找满足条件的，与SGI是相反的

```c
template<class _FI1, class _FI2> inline
	_FI1 search(_FI1 _F1, _FI1 _L1, _FI2 _F2, _FI2 _L2)//[意思是看F1到L1之间是否存在F2之L2完全匹配的子序列，
//若存在则返回F1否则返回L1]
	{return (_Search(_F1, _L1, _F2, _L2,
		_Dist_type(_F1), _Dist_type(_F2))); }
template<class _FI1, class _FI2, class _Pd1, class _Pd2> inline
	_FI1 _Search(_FI1 _F1, _FI1 _L1, _FI2 _F2, _FI2 _L2,
		_Pd1 *, _Pd2 *)
	{_Pd1 _D1 = 0;
	_Distance(_F1, _L1, _D1);
	_Pd2 _D2 = 0;
	_Distance(_F2, _L2, _D2);
	for (; _D2 <= _D1; ++_F1, --_D1)
		{_FI1 _X1 = _F1;
		for (_FI2 _X2 = _F2; ; ++_X1, ++_X2)
			if (_X2 == _L2)
				return (_F1);
			else if (!(*_X1 == *_X2))
				break; }
	return (_L1); }
    ```

五：STL之for_each（）算法
====

   主要用到的就是仿函数的思想以及模板实参推演技术    
   ```c
Template<class  T>
Struct  Print
{
   Operator()(const T&x)
{
    Cout<<x<<”  ”;
}
};

Template<class IT,class op>
For_each(IT I,IT L ,op fn)
{
for(;I!=L;++I)
{
  fn(*I);
}
}、
```

六：STL的容器
=====

* 1：首先STL的所有关联容器都自动拥有排序功能（由于底层采用的是红黑树）所以不需要用到这个sort算法，
对于序列容器stack  queue  priority_queue由于拥有特定的出入口，所以不允许对其排序，剩下的对于
vector  deque  list 前两者属于随机迭代器适合使用sort而list自己拥有sort排序方法  

* 2:vector底层使用的是数据组实现的，stack和queue底层使用的是list[是双向迭代器]    

* 3：注意list与vector的clear函数的区别，首先vector中的clear 只是把finish修改而且调用了析构函数
但是没有释放空间，但是list中的clear却不但要调用析构函数而且还要释放空间（其主要原因是：由于
vector是连续空间 而 list却不是连续的空间所以每当删除某一个节点的时候就要释放其空间不然就会造
成内存泄露）     

* 4：注意stack以及queue  priority_queue没有迭代器     


七：绑定器的介绍
======
   首先绑定器分为一元绑定器与二元绑定器   
   一元绑定器：就是把函数的第一个参数和函数绑定起来    
   二元绑定器：就是把函数的第二个参数与函数绑定起来   

实现方法就是：以下是二元绑定器实例：  
``` c
#if 0
template<class _A1, class _A2, class _R>
struct binary_function 
{
	typedef _A1 first_argument_type;
	typedef _A2 second_argument_type;
	typedef _R  result_type;
};

template<class _Ty>
struct less : binary_function<_Ty, _Ty, bool> 
{
	bool operator()(const _Ty& _X, const _Ty& _Y) const
	{
		return (_X < _Y); 
	}
};
#endif
//////////////////////////////


template<class _Bfn>
struct  binder2nd :
       public binary_function<_Bfn::first_argument_type,
		                      _Bfn::second_argument_type,
							  _Bfn::result_type>
{
	_Bfn op;
	_Bfn::second_argument_type value;
public:
	binder2nd(const _Bfn &_X,
		const _Bfn::second_argument_type &_Y)
		:op(_X),value(_Y)
	{}
	_Bfn::result_type operator()(_Bfn::first_argument_type &_X)
	{
		return op(_X,value);
	}
};

template<class _Bfn,class _Ty> [//此步是为了解决二货的人给的如：binder2nd<less<int>> (less<double>(),4)  两个类型不一致
]
binder2nd<_Bfn> bind2nd(const _Bfn &_X,const _Ty &_Y)//用binder2nd<_Bfn> 定义对象bind2nd
{
	return binder2nd<_Bfn>(_X,(_Bfn::second_argument_type)_Y);

}//   binder2nd<less<int>> (less<int>(),4)     bind2nd(less<int>(),4)

template<class _Ty,class _Bfn>
void Show(_Ty *ar,int n,_Bfn _Fn)
{
	for(int i=0;i<n;++i)
	{
		if(_Fn(ar[i]))
		{
			cout<<ar[i]<<"  ";
		}
	}
	cout<<endl;
}
void main()
{
	int ar[10]={1,2,3,4,5,6,7,8,9,10};
	greater<int> gt;
	less<int> ls;
	Show(ar,10,bind2nd(less<int>(),4));
}
```
八：STL各种容器的比较
======
* 1 vector  
向量 相当于一个数组   
注意： 
  _First是指指向数组的第一个元素   
  _Last是指有效元素的下一个位置     
  _End是指所有空间最后的下一个位置     
    在内存中分配一块连续的内存空间进行存储。支持不指定vector大小的存储。STL内部实现时，
首先分配一个非常大的内存空间预备进行存储，即capacituy（）函数返回的大小，当超过此分配
的空间时再整体重新放分配一块内存存储，这给人以vector可以不指定vector即一个连续内存的大
小的感觉。通常此默认的内存分配能完成大部分情况下的存储。     
  * 优点：
         (1) 不指定一块内存大小的数组的连续存储，即可以像数组一样操作，但可以对此数组 
进行动态操作。通常体现在push_back() pop_back()
         (2) 随机访问方便，即支持[ ]操作符和vector.at()
         (3) 节省空间。
 *  缺点：
         (1) 在内部进行插入删除操作效率低。
         (2) 只能在vector的最后进行push和pop，不能在vector的头进行push和pop。
         (3) 当动态添加的数据超过vector默认分配的大小时要进行整体的重新分配、拷贝与释放
                     
* 2 list
    双向链表    
    每一个结点都包括一个信息快Info、一个前驱指针Pre、一个后驱指针Post。
可以不分配必须的内存大小方便的进行添加和删除操作。使用的是非连续的内存
空间进行存储   。   
 *  优点：   
         (1) 不使用连续内存完成动态操作。  
         (2) 在内部方便的进行插入和删除操作   
         (3) 可在两端进行push、pop   
 *  缺点：
         (1) 不能进行内部的随机访问，即不支持[ ]操作符和vector.at()  
         (2) 相对于verctor占用内存多  

* 3 deque
   双端队列 double-end queue  
   deque是在功能上合并了vector和list。   
  * 优点：    
         (1) 随机访问方便，即支持[ ]操作符和vector.at()    
         (2) 在内部方便的进行插入和删除操作   
         (3) 可在两端进行push、pop   
  * 缺点：    
        (1) 占用内存多   

使用区别：   
     1 如果你需要高效的随即存取，而不在乎插入和删除的效率，使用vector    
     2 如果你需要大量的插入和删除，而不关心随即存取，则应使用list    
     3 如果你需要随即存取，而且关心两端数据的插入和删除，则应使用deque    



九：STL总的概述
======

STL提供六大组件，彼此可以组合套用   

* 1、容器（containers）：各种数据结构，如vertor，list，deque，set，map.从实现的角度来看，
STL容器是一种class template    

* 2、算法（algorithms）：各种算法如sort，search，copy，earse。STL算法是一种 function template。

* 3、迭代器（iterators）：扮演容器与算法之间的胶合剂，是所谓的“泛型指针”。所有STL容器都有自己
的专属的迭代器。   

* 4、仿函数（functors）：行为类似函数，可以作为算法的某些策略。从实现的角度来看，仿函数是一种
 重载了operator()的class或class template。    

* 5、配接器（adapters实际上是一种设计模式）[即将一个class的接口转化为另一个class的接口，使其原
 来不兼容的接口变成可以合作的；  

通常使用两种方法可以完成

* 方法一：
   增加一个daapter类 ，然后使其公有继承类1私有继承类2，然后在重写类1的方法    

* 方法二：
   增加一个daapter类 ，然后使其公有继承类1，把类2作为自己的子对象，然后在
重写类1的方法里面调用类2的新方法   

]：一种用来修饰容器或仿函数或迭代器借口的东西。例如queue和stack      

* 6、配置器（allocators）：负责空间的配置与管理。配置器是一个实现了动态空间分配、空间管理、
空间释放的class template。   


欢迎一起交流　　　　
====
 
在使用中有任何问题，欢迎反馈给我，可以用以下联系方式跟我交流

* 邮件(1031379296#qq.com, 把#换成@)
* QQ: 1031379296
* weibo: [@王发康](http://weibo.com/u/2786211992/home)


Thx
====

* chunshengsterATgmail.com


Author
====
* Linux\nginx\golang\c\c++爱好者
* 欢迎一起交流  一起学习# 
* Others say good and Others good
