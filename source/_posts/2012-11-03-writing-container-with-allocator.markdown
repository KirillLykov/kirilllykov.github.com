---
layout: post
title: "Writing container with allocator"
date: 2012-11-03 19:08
comments: true
categories: 
---
In this post I will write how to implement container using allocators. As an example of container to be developed I picked up
2D array of elements.
<!-- more -->
<h2>Tests for container</h2>

I firmly believe that tests should be developed before writing any code, especially general-purpose library code.
Lets call our container grid2D. It will be template class which has two template parameters - data type and allocator type.
By default allocator type will be std::allocator.
I've chousen google c++ test framework. It is pretty easy to use. First, download it, then build it using make or cmake.
Cmake is preferable, but it can also be build in old-fashion maner:
```
./configure
make
```
Built libraries can be found in  ${GTEST_ROOT}/lib/.lib. Show them to your linker and add to include directories ${GTEST_ROOT|/include.
If linker says that it can't find gtest library (it may try to find them somewhere in a system directory), copy it to the place where 
linker is looking it. 
All test functions should be declared using TEST macros. The syntax is simple: TEST(<test_group_name>, \<test_name\>) { ... }
Google test frameworks employes several macroses to check test\'s conditions. The most useful is EXPECT_EQ(arg1, arg2) which
expects equaty of arg1 and arg2. If the condition is not sutisfied, it will not terminate the process.
Test functions might be called from main function.
```
#include <gtest/gtest.h>

TEST(TwoDGridTest, construction)
{
    grid2D<double> grid(1, 2);
}

TEST(TwoDGridTest, copyConstruction)
{
    grid2D<double> grid1(1, 2);
    grid2D<double> grid2(grid1);
    EXPECT_EQ(grid1, grid2);
}

TEST(TwoDGridTest, assignment)
{
    grid2D<double> grid1(1, 2);
    grid2D<double> grid2(1, 2);
    grid2 = grid1;
    EXPECT_EQ(grid1, grid2);
}

// grid with differnt sizes must not be assigned
TEST(TwoDGridTest, assignmentBrake)
{
  grid2D<double> grid1(1, 2);
  grid2D<double> grid2(2, 3);
  try {
    grid2 = grid1;
  }
  catch (const AllocationUtils::array_size_error& er)
  {
    return;
  }
  EXPECT_TRUE(false);
}

TEST(TwoDGridTest, access)
{
  size_t n = 2, m = 4;
  grid2D<double> grid(n, m);
  for (size_t i = 0; i < n; ++i) {
    for (size_t j = 0; j < m; ++j) {
      grid(i, j) = i * j;
    }
  }

  const grid2D<double> grid2 = grid;
  for (size_t i = 0; i < n; ++i) {
    for (size_t j = 0; j < m; ++j) {
      EXPECT_EQ(grid2(i, j), i * j);
    }
  }
}

int main(int argc, char** argv)
{
  ::testing::InitGoogleTest(&argc, argv);
  return RUN_ALL_TESTS();
}

```
I hope test above are self-explanatory.

<h2>Implementing container</h2>
Why do we need to use allocator? Because allocation of elements in the container may be done in several ways.
The most common case in implemented in std::allocator but this implementation is not acceptable if your container is 
used by several threads, processes or when you prefer to use your own memory manager. Examples of different allocators
implementation can be found in boost: boost::mpi::allocator, boost::interprocess::allocator, boost::detail::quick_allocator.

Grid class should have two templates parameters - type of objects to be stored and allocator type. I use 
nested template for allocator because it gives additional flexibility in comparison with class allocator = std::allocator<T>.
Grid has sizes, pointer to the block of data, and allocator instance. Allocator is declared mutable because I want to use it in
const methods while allocators methods are not const.
```
  template< class T, template<typename X> class allocator = std::allocator >
  class grid2D
  {
    size_t m_n1, m_n2;
    mutable allocator<T> m_allocator;
    size_t m_linearSz;
    T* m_data;
    
    typedef grid2D<T, allocator> _TGrid;
  public:
    grid2D(size_t n1, size_t n2)
       :  m_n1(n1), m_n2(n2), m_linearSz(n1 * n2), m_data(0)
     {
       m_data = m_allocator.allocate(m_linearSz);
       std::uninitialized_fill_n(m_data, m_linearSz, T());
     }

    grid2D(const _TGridImpl& another)
      : m_n1(another.m_n1), m_n2(another.m_n2), m_linearSz(another.m_linearSz)
    {
      m_data = m_allocator.allocate(m_linearSz);
      std::uninitialized_copy(another.m_data, another.m_data + m_linearSz, this->m_data);
    }
```
In constructors, I allocate linear block of memory and then call functions with preffix uninitialized which means that they work with allocated but not initialized memory (constructos were not called).
It is important to underline that method allocator::allocate does not call constructors for the objects.
Call of std::uninitialized_fill_n will call constructors for the allocated blocks of memory. It is implemented like that:
```
  template<typename _T1>
  inline void construct(_T1* p)
  {
    ::new(static_cast<void*>(p)) _T1;
  }

  template<typename _ForwardIterator, typename _Size>
  void uninit_fill_n(_ForwardIterator first, _Size n)
  {
    _ForwardIterator cur = first;
    try
    {
      for (; n > 0; --n, ++cur)
        construct(addressof(*cur));
    }
    catch (...)
    {
      destroy(first, cur);
      throw;
    }
  }
```
Similarly, algorithm std::uninitialized_copy calls copy-constructors without allocating memory for objects. So we see that allocator separates raw memory allocation and 
calls of constructos. It was done for the sake of flexibility, efficiency and safity. //some explanations here


```
    virtual ~grid_impl()
    {
      destroy(m_data, m_data + m_linearSz);
      m_allocator.deallocate(m_data, m_linearSz);
    }
```


The next crucial method is operator= in which I employ deep-copy approach - i.e just copy all object from the source grid into the target using std::copy:
```
    _TGrid& operator= (const _TGrid& another)
    {
      if (this == &another)
        return *this;

      if (m_n1 != another.m_n1 || m_n2 != another.m_n2)
        throw logical_error();

      std::copy(another.m_data, another.m_data + m_linearSz, this->m_data);

      return *this;
    }
```

Additionally, we may add operator== and operator!= which use std::equal algorithm:
```
    bool operator== (const _TGridImpl& another) const
    {
     return std::equal(m_data, m_data + m_linearSz, another.m_data);
    }

    bool operator!= (const _TGridImpl& another) const
    {
     return !this->operator== (another);
    }
```

Basicly destroy is implemeted this way in std::alocator:
```
  template<typename T>
  inline void destroy(T* p) { p->~T(); }

  template<typename _ForwardIterator>
  void destroy(_ForwardIterator first, _ForwardIterator last)
  {
    for (; first != last; ++first)
      destroy(addressof(*first));
  }
```

Finally, the most important operator is access to the elements. It can be implemented in form get(size_t i, size_t j)
but I prefer a bit more sugarish implementation with operator():
```
    const T& operator() (size_t i, size_t j) const
    {
      return _TGridImpl::m_data[i + m_n1 * j];
    }

    T& operator() (size_t i, size_t j)
    {
      return _TGridImpl::m_data[i + m_n1 * j];
    }
```
