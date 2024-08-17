# Cpp中一些函数、API的使用

## 一、文件I/O相关

### 1.1 读写文件：fopen

`fopen` 系列函数是 **C 标准库**中的文件操作 API，用于打开、读取、写入和关闭文件。这些函数为文件操作提供了简单的接口。在使用时需要**<font color='red'>包含`<stdio.h>`头文件 。</font>**

- `fopen`函数：打开文件，返回一个指向 `FILE`结构的指针，这个指针用于后续的文件操作。	

  ```cpp
  FILE *fopen(const char *filename, const char *mode);	
  
  // filename: 文件名的字符串，指定要打开的文件路径。
  // mode: 文件打开模式，用于指定文件的打开方式（如只读、只写、追加等）。
  // "r": 以只读模式打开文件。文件必须存在。
  // "w": 以只写模式打开文件。如果文件不存在，则创建该文件；如果文件已存在，则清空文件内容。
  // "a": 以追加模式打开文件。如果文件不存在，则创建该文件；如果文件已存在，则将数据追加到文件末尾。
  // "r+": 以读写模式打开文件。文件必须存在。
  // "w+": 以读写模式打开文件。如果文件不存在，则创建该文件；如果文件已存在，则清空文件内容。
  // "a+": 以读写模式打开文件。如果文件不存在，则创建该文件；如果文件已存在，数据将被追加到文件末尾。
  ```

- `fgets`函数：从文件中读取一行，存储到指定的缓冲区中。

  ```cpp
  char* fgets(char *str, int n, FILE *stream);
  
  // str: 指向存储读取数据的字符数组的指针。
  // n: 要读取的字符数（包括终止符 \0）。fgets 最多读取 n-1 个字符，并自动在末尾添加 \0 终止符。
  // stream: 指向已打开的文件的 FILE 指针。
  // 返回 str 指针，如果遇到文件结束或发生错误，则返回 NULL。
  ```

- `fwrite`函数：将数据写入到文件。

  ```cpp
  size_t fwrite(const void *ptr, size_t size, size_t count, FILE *stream);
  
  // ptr: 指向要写入的数据的指针。
  // size: 每个数据项的大小，以字节为单位。
  // count: 要写入的项数。
  // stream: 指向已打开的文件的 FILE 指针。
  // 返回值是成功写入的项数。如果返回值小于 count，则可能发生了错误。
  ```

- `fread` 函数：用于从文件中读取数据。

  ```cpp
  size_t fread(void *ptr, size_t size, size_t count, FILE *stream);
  ```

  - `ptr`: 指向存储读取数据的缓冲区。
  - `size`: 每个数据项的大小，以字节为单位。
  - `count`: 要读取的项数。
  - `stream`: 指向已打开的文件的 `FILE` 指针。

  - 返回值是成功读取的项数。如果返回值小于 `count`，则可能遇到文件结尾或发生了错误。

- `fseek` 函数：用于移动文件指针到指定位置。

  ```cpp
  int fseek(FILE *stream, long offset, int whence);
  ```

  - `stream`: 指向已打开的文件的 `FILE` 指针。
  - `offset`: 偏移量，以字节为单位。
  - `whence`:  参考位置，可以是以下值之一：
    - `SEEK_SET`: 文件开头。
    - `SEEK_CUR`: 当前文件指针位置。
    - `SEEK_END`: 文件末尾。

  - 返回值为 `0` 表示成功，`非零` 表示失败。

- `fclose` ：用于关闭文件并释放与文件相关的资源。

  ```cpp
  int fclose(FILE *stream);
  ```

  - `stream`: 指向要关闭的文件的 `FILE` 指针。

  - `fclose` 成功关闭文件时返回 `0`，如果发生错误，返回 `EOF`。

### 1.2 文件I/O流：fstream、ifstream、ofstream

是 C++ 标准库中用于文件 I/O 的流类，可以用于读写文件。

- `ifstream` 用于从文件中读取数据，它继承自 `istream`，专门处理文件输入操作。
- `ofstream` 用于向文件中写入数据，它继承自 `ostream`，专门处理文件输出操作。
- `fstream` 继承自 `iostream`，并结合了 `ifstream` 和 `ofstream` 的功能，能够同时处理输入（读）和输出（写）操作。

```c++
#include <fstream>

int main() {
    std::fstream file;
    // 读写模式打开文件
    file.open("example.txt",  std::ios::in | std::ios::out | std::ios::app);
    
    // 逐行读取文件内容
    while (std::getline(file, line)) {	    
        std::cout << line << std::endl;
    }
    
    // 写入文件
    file << "hellow world!" << std::endl;
    
    // 关闭文件
    file.close();	
}
```

常见的模式有：

- `std::ios::in`：读模式（输入）
- `std::ios::out`：写模式（输出）
- `std::ios::binary`：二进制模式
- `std::ios::app`：追加模式
- `std::ios::ate`：初始位置在文件末尾
- `std::ios::trunc`：如果文件存在，清空文件内容





## 二、多线程、锁

### 2.1 C语言线程库`pthread`（POSIX threads）

#### 2.2.1 线程创建 `pthread_create`

```cpp
#include <pthread.h>

pthread_t thread;
ThreadData args = {1, "Hello from parameterized thread"};
int result = pthread_create(&thread, attr, function, args);		// 线程创建即启动。
```

- `arttr`：定制各种不同的线程属性，通常直接设为NULL；
- `function`：线程要执行的函数；
- `args`：函数执行需要输入的参数，无参数是输入NULL，有参数时需要输入`ThreadData`结构体对象。
- 返回值：成返回0，失败返回非0值。

#### 2.2.2 线程同步

- **互斥锁：`pthread_mutex_t`**

  ```cpp
  // 静态初始化互斥锁
  pthread_mutex_t m_mutex = PTHREAD_MUTEX_INITIALIZER;
  // 动态初始化互斥锁
  pthread_mutex_t m_mutex;
  pthread_mutex_init(&m_mutex, nullptr);
  // 加锁
  pthread_mutex_lock(&m_mutex);
  // 解锁
  pthread_mutex_unlock(&m_mutex);
  // 销毁
  pthread_mutex_destroy(&m_mutex);
  ```

- **条件变量：`pthread_cond_t`**

  ```cpp
  // 静态初始化条件变量
  pthread_cond_t m_cond = PTHREAD_COND_INITIALIZER;
  // 动态初始化条件变量
  pthread_cond_t m_cond;
  pthread_cond_init(&m_cond, nullptr);
  // 阻塞
  pthread_cond_Wait(&m_cond, &m_mutex);	// 要同时输入一个互斥锁对象，代表是等待申请该互斥锁
  // 唤醒一个其他等待线程
  pthread_cond_signal(&m_mcond);
  // 唤醒全部其他等待线程
  pthread_cond_broadcast(&m_mcond);
  ```

#### 2.2.3 等待线程结束 `pthread_join`

```cpp
#include <pthread.h>
int pthread_join(pthread_t thread, void **retval);
```

- `thread`：要等待结束的线程；
- `retval`：若不为NULL，则用于复制出线程的退出状态；
- 返回值：成功返回0，失败返回非0。

#### 2.2.4 线程分离 `pthread_detach`

```cpp
#include <pthread.h>
int pthread_detach(pthread_t thread);
```

- `thread`：要分离的线程；
- 返回值：成功返回0，失败返回非0。

#### 2.2.5 线程退出 `pthread_exit`

```cpp
#include <pthread.h>
void pthread_exit(void *retval);
```

- `retval`：用于保存线程的退出状态；

### 2.2 C+11标准库的线程 `std::thread`

#### 2.2.1 线程创建 `std::thread`

```cpp
#include <thread>
// 使用函数
std::thread t(func, args1, std::ref(args2), std::cref(args3));		// 线程创建即启动。
// 使用lambda表达式
std::thread t([]() { std::cout << "hellow, world!" << std::endl;});  // 线程创建即启动。
// 使用类成员函数
std::thread t(&MyClass::func, &obj, args1, args2, ...);		// obj是实际依托的对象
// 使用类静态成员函数不需要传递对象
std::thread t(&MyClass::func, args1, args2, ...);	
```

- `std::ref()`：指传递引用参数；
- `std::cref()`：指传递常量引用参数。

#### 2.2.2 线程同步

- **互斥锁： `std::mutex` **

  ```cpp
  #include <mutex>
  // 初始化
  std::mutex m_mtx;
  // 加锁
  m_mtx.lock();
  // 解锁
  m_mtx.unlock();
  ```

- **RAII（资源获取即初始化）锁： `std::lock_guard` **

  用于**管理某个锁(Lock)对象**，因此与 Mutex RAII 相关，**方便线程对互斥量上锁**，即在某个 lock_guard 对象的声明周期内，它所管理的锁对象会一直保持上锁状态；而 lock_guard 的生命周期结束之后，它所管理的锁对象会自动解锁 (注：类似 shared_ptr 等智能指针管理动态分配的内存资源 )。

  > lock_guard 对象不负责管理 Mutex 对象的生命周期，只是简化了 Mutex 对象的**上锁和解锁**操作
  >
  > 简单理解就是自动unlock

  ```c++
  #include<mutex>
  // 初始化
  std::mutex m_mtx;
  
  void func() {
  	std::lock_guard<std::mutex> lc(m_mtx);		// 加锁
  	...
  }
  // 函数执行完毕lc资源释放后会自动解锁，无需显式的unlock
  ```

- **RAII锁： `std::unique_lock` **

  更灵活的锁，它**允许手动锁定和解锁互斥量**，以及与条件变量一起使用（是lock_guard的进阶版）。与 lock_guard 类似，unique_lock 也是一个 RAII 风格的锁，当对象离开作用域时，它会自动解锁互斥量。unique_lock 还**支持延迟锁定、尝试锁定和可转移的锁所有权**。

  ```cpp
  #include<mutex>
  // 初始化
  std::mutex m_mtx;
  
  void func() {
  	std::unique_lock<std::mutex> lc(m_mtx);		// 加锁
  	...
  }
  // 函数执行完毕lc资源释放后会自动解锁，无需显式的unlock
  ```

- **条件变量：`std::condition_variable`**

  ```cpp
  #include <condition_variable>
  #include <mutex>
  
  std::mutex m_mtx;
  std::unique_lock<std::mutex> lc(m_mtx);
      
  // 初始化 
  std::condition_variable cv;
  // 阻塞
  cv.wait(lc);
  // 唤醒一个阻塞线程
  cv.notify_one();
  // 唤醒全部阻塞线程
  cv.notify_all();
  ```

#### 2.2.3 等待线程结束  `thread.join()`

```cpp
#include <thread>

std::thread t(func);
t.join();	// 等待线程结束
```

#### 2.2.4 线程分离 `thread.detach()`

```
#include <thread>

std::thread t(func);
t.detach();	// 等待线程结束
```

#### 2.2.5 线程退出

C++标准库std::thread并没有提供类似`pthread_exit`函数相关显式退出线程的函数，原因是 `std::thread` 的设计是面向 RAII（Resource Acquisition Is Initialization）原则的，即资源管理应当通过对象的生命周期来控制。

尽管没有 `pthread_exit`，你仍然可以通过控制线程的执行逻辑来实现类似的功能。如，通过在线程函数中**检查某个退出条件或标志位**，在满足条件时退出函数。

```cpp
#include <iostream>
#include <thread>
#include <atomic>
#include <chrono>

std::atomic<bool> stop_thread(false);

void worker() {
    while (!stop_thread) {
        std::cout << "Working..." << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
    }
    std::cout << "Worker thread is exiting..." << std::endl;
}

int main() {
    std::thread t(worker);
    std::this_thread::sleep_for(std::chrono::seconds(2));
    stop_thread = true;  // Signal the thread to exit
    t.join();
    std::cout << "Main thread is done." << std::endl;
    return 0;
}
```

