---
title: "Concurrency for cracked ones."
date: 2025-09-21
draft: false
---

## What is thread?

lightweight process

## When to use thread??

- heavy computation
- to separate work

## Modern cpp Concurrency

- std::thread
- execute simple single thread eg:

```cpp
#include <iostream>
#include <thread>
using namespace std;

void test(int x) {
  cout << "Hello frm test\n";
  cout << "arguments" << x << "\n";
}
int main() {
  thread myThread(&test, 100);
  cout << "Hello from main \n";
  return 0;
}
```

- lamdafucntion: define fuction in short form witin a line

```cpp
#include <iostream>
#include <thread>
using namespace std;

int main() {

  auto lambda = [](int x, int y) {
    cout << "Hello frm test\n";
    cout << "arguments" << x << y << "\n";
  };
  cout << "Hello from main \n";
  thread myThread(lambda, 100, 20);
  myThread.join();
  return 0;
}

```

- multithread execute multiple thread saving them in vector

```cpp
#include <iostream>
#include <thread>
#include <vector>
using namespace std;

int main() {

  auto lambda = [](int x) {
    cout << "Hello frm test" << this_thread::get_id() << "\n";
    cout << "index" << x << "\n";
  };
  vector<thread> threads;
  for (int i = 0; i < 5; ++i) {
    threads.push_back(thread(lambda, i));
  }
  for (int i = 0; i < 5; ++i) {
    threads[i].join();
  }
  cout << "Hello from main \n";
  return 0;
}

```

- jthread supported after cpp 20 it joins the thread automatically meaning make main thread wait until other finishes

```cpp
// c++ 20
#include <iostream>
#include <thread>
#include <vector>
using namespace std;

int main() {

  auto lambda = [](int x) {
    cout << "Hello frm test" << this_thread::get_id() << "\n";
    cout << "index" << x << "\n";
  };
  vector<jthread> jthreads;
  for (int i = 0; i < 5; ++i) {
    jthreads.push_back(jthread(lambda, i));
  }
  cout << "Hello from main \n";
  return 0;
}

```

- racecondition example

```cpp
// racecodition is due to the context switching
#include <iostream>
#include <thread>
#include <vector>
using namespace std;
static int shared_value = 0;
auto increment = []() { shared_value += 1; };
int main() {
  vector<thread> threads;
  for (int i = 0; i < 5000; ++i) {
    threads.push_back(thread(increment));
  }
  for (int i = 0; i < 5000; ++i) {
    threads[i].join();
  }
  cout << shared_value << "\n";
  return 0;
}

```

```bash
100
~/concurrency$ ./a.out
100
~/concurrency$ ./a.out
100
~/concurrency$ ./a.out
99(inconsistency)
~/concurrency$ ./a.out
100
~/concurrency$ ./a.out
100
~/concurrency$ ./a.out
100

```

- solution mutex(mutual exclusion) also called binary semaphore

```cpp
#include <iostream>
#include <mutex>
#include <thread>
#include <vector>

std::mutex gLock;
using namespace std;
static int shared_value = 0;
auto increment = []() {
  gLock.lock();
  shared_value += 1;
  gLock.unlock();
};
int main() {
  vector<thread> threads;
  for (int i = 0; i < 100; ++i) {
    threads.push_back(thread(increment));
  }
  for (int i = 0; i < 100; ++i) {
    threads[i].join();
  }
  cout << shared_value << "\n";
  return 0;
}

```

- deadlock lock was never released now no thread is able to make progress

```cpp
#include <iostream>
#include <mutex>
#include <thread>
#include <vector>

std::mutex gLock;
using namespace std;
static int shared_value = 0;
auto increment = []() {
  gLock.lock();
  try {
    shared_value += 1;
    throw "danger";
  } catch (...) {
    cout << "exception";
    return;
  }
  gLock.unlock();
};
int main() {
  vector<thread> threads;
  for (int i = 0; i < 100; ++i) {
    threads.push_back(thread(increment));
  }
  for (int i = 0; i < 100; ++i) {
    threads[i].join();
  }
  cout << shared_value << "\n";
  return 0;
}

```

- solution lock_guard mutex wrapper that provides convenient RAII-style mechanism (Resource Allocation is initiallized)

```cpp
#include <iostream>
#include <mutex>
#include <thread>
#include <vector>

std::mutex gLock;
using namespace std;
static int shared_value = 0;
auto increment = []() {
  lock_guard<mutex> lockGuard(gLock);
  shared_value += 1;
};
int main() {
  vector<thread> threads;
  for (int i = 0; i < 100; ++i) {
    threads.push_back(thread(increment));
  }
  for (int i = 0; i < 100; ++i) {
    threads[i].join();
  }
  cout << shared_value << "\n";
  return 0;
}

```

- atomic to update shared value better way than mutex

```cpp
#include <atomic>
#include <iostream>
#include <thread>
#include <vector>

using namespace std;
static atomic<int> shared_value = 0;
auto increment = []() { shared_value++; };
int main() {
  vector<thread> threads;
  for (int i = 0; i < 100; ++i) {
    threads.push_back(thread(increment));
  }
  for (int i = 0; i < 100; ++i) {
    threads[i].join();
  }
  cout << shared_value << "\n";
  return 0;
}

```

- mutex was necessary cause if a datatype is greater than the word atomic can't be implemented

## Modern cpp parallelism

- if data accessed by different thread are independent of each other we don't need to lock thing up while supports parallelism example below where two section of array is divided which is accessed by different threads and computed partial sum

```cpp
#include <iostream>
#include <thread>
#include <vector>

using namespace std;

// Function to compute partial sum
void partial_sum(const vector<int> &arr, int start, int end,
                 long long &result) {
  long long sum = 0;
  for (int i = start; i < end; i++) {
    sum += arr[i];
  }
  result = sum;
}

int main() {
  vector<int> arr(1000000, 1); // 1 million elements, all = 1

  long long result1 = 0, result2 = 0;

  thread t1(partial_sum, cref(arr), 0, arr.size() / 2, ref(result1));
  thread t2(partial_sum, cref(arr), arr.size() / 2, arr.size(), ref(result2));

  t1.join();
  t2.join();

  long long total = result1 + result2;

  cout << "Total sum = " << total << endl;
  return 0;
}

```

## Condition Variable

- when a thread say thread1 is accessing critical section if used mutex other threads are constantly checking if thread is unlocked which wasting a lot of cpu cycle(spin lock) so a better way Condition variable comes to play example

```cpp
#include <chrono>
#include <condition_variable>
#include <iostream>
#include <mutex>
#include <thread>
using namespace std;

mutex gLock;
condition_variable gCV;
int main() {
  int result = 0;
  bool notify = false;
  // reporting thread
  // Mut wait on work
  thread reporter([&] {
    unique_lock<mutex> lock(gLock);
    if (!notify) {
      gCV.wait(lock);
    }
    cout << "Reported,result is:" << result << "\n";
  });
  // worker thread
  thread worker([&] {
    unique_lock<mutex> lock(gLock);
    // do work cause we have lock
    result = result + 43 + 1;
    // our work is done so notify
    notify = true;
    this_thread::sleep_for(chrono::seconds(5));
    cout << "Work done baby" << "\n";
    gCV.notify_one();
  });
  worker.join();
  reporter.join();
  return 0;
}
```

Note:

```txt
The square brackets [] tell the lambda what variables from the outer scope it can use.

[ ] → captures nothing.

[=] → captures all outer variables by value (makes a copy).

[&] → captures all outer variables by reference (no copy, uses them directly).

[x] → captures only x by value.

[&x] → captures only x by reference.
```

- requires a little bit of extra effort but very powerful requires
  1.boolean(notify)
  2.lock uniquelock(The class unique_lock is a general-purpose mutex ownership wrapper allowing deferred locking, time-constrained attempts at locking, recursive locking, transfer of lock ownership, and use with condition variables.)
  3.Condition variable
  4.2 threads worker/reporter

## How many thread can i fire?

- **Operating System Limits**
  - Each OS sets a maximum thread count per process.
    - **Linux**: limited by available virtual memory and the kernel’s `ulimit -u` (max user processes). Each thread usually consumes ~8 MB of stack space by default, so thousands are possible if memory allows.
    - **Windows**: typically supports thousands, but again limited by memory.

- **Available Memory**
  - Each thread needs stack memory. Default is often:
    - **Linux (glibc pthreads)** → 8 MB per thread
    - **Windows** → ~1 MB per thread
  - Example: On a system with 8 GB RAM, if each thread uses 1 MB stack, _in theory_ you could create ~8000 threads. In practice, other processes and overhead reduce this.

- **CPU Cores vs Threads**
  - Having more threads than CPU cores is fine (they get time-sliced), but too many can lead to heavy context-switching overhead.
  - As a rule of thumb:
    - For **CPU-bound work** → number of threads ≈ number of cores (or cores ± 1).
    - For **I/O-bound work** → you can have many more threads, since most will be waiting.

- **Practical Numbers**
  - **Lightweight workloads**: tens of thousands of threads may be possible (but performance may drop).
  - **Real-world usage**: typically a few hundred threads max. For larger scales, use **thread pools** or **async/event-driven models** instead.

## Async and future

- Function runned async runs on separate thread and we don't wait for it's completion until get function is called for future value.
- Some useful usecase is that loading data in bg when running something and that data will be used in future eg youtube is loading video(grey area) while playing.

```cpp
#include <future>
#include <iostream>
using namespace std;
int square(int x) { return x * x; }
int main() {
  future<int> asyncFunc = async(&square, 9);
  // as function is called aync we can do other task below and it will not wait
  // for it's completion unti get is called line below
  for (int i = 0; i < 10; ++i) {
    cout << square(i) << "\n";
  }
  cout << asyncFunc.get() << "\n";
  return 0;
}

```

## Thread Sanitizer

- Use to detect data race so toolkit for writing better threaded program
- It is implemented using shadow memory(which stores metadata about each byte/word) on using TSan sanitizer checks shadow memory and finds if race or not

```bash
    g++ -fsanitize=thread racecondition.cpp
```

## Try lock

- Try lock idea is that the thread trys to lock a data if it doesn't get lock,it will not wait for the lock it will continue doing something else apart from staying in waiting loop
- When can this idea be useful
  . It can be useful if thread can do something userful with their cpu cycle rather than waiting example if multiple producer threads are producing and storing them in buffer if one trys and don't get the lock it can produce other data for future and attempt to get lock in sencond try example below
- try to use RAII-style using adopt_lock

```cpp
#include <chrono>
#include <iostream>
#include <mutex>
#include <thread>

using namespace std;

mutex gLock;

void Job1() {
  if (gLock.try_lock()) {
    cout << "Job1 executed \n";
    gLock.unlock();
  }
}

void Job2() {
  if (gLock.try_lock()) {
    cout << "Job2 executed \n";
    gLock.unlock();
  } else {
    // May be do something useful
    this_thread::sleep_for(chrono::milliseconds(200));
    if (gLock.try_lock()) {
      cout << "Job 2 executed in 2nd try";
      gLock.unlock();
    }
  }
}

int main() {
  thread thread1(&Job1);
  thread thread2(&Job2);

  thread1.join();
  thread2.join();
  return 0;
}

```

## Semaphore() (counting)

-A counting_semaphore is a lightweight synchronization primitive that can control access to a shared resource. Unlike a std::mutex, a counting_semaphore allows more than one concurrent access to the same resource, for at least LeastMaxValue concurrent accessors. The program is ill-formed if LeastMaxValue is negative.

- two types in cpp counting and binary under header <semaphore> instentiated with first argument of maxthread that could access the shared memory
- Note implemented in cpp 20 and above

```cpp
#include <iostream>
#include <semaphore>
#include <thread>
#include <vector>

using namespace std;
counting_semaphore gSl(2);

void Work(int id) {
  cout << "Thread " << id << " waiting...\n";
  gSl.acquire();
  cout << "Thread " << id << " acquired!\n";

  this_thread::sleep_for(1s);

  cout << "Thread " << id << " releasing.\n";
  gSl.release();
}
int main() {
  vector<thread> threads;
  for (int i = 0; i < 21; ++i) {
    threads.push_back(thread(Work, i));
  }
  for (int i = 0; i < 21; ++i) {
    threads[i].join();
  }
  return 0;
}

```

## Reference

[Modern Concurrency Mike Shah](https://www.youtube.com/playlist?list=PLvv0ScY6vfd_ocTP2ZLicgqKnvq50OCXM)

[cpp reference](https://www.cppreference.com/w/cpp/atomic.html)

[code](https://github.com/Aashish1-1-1/concurrency)
