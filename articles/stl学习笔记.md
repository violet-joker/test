# STL容器

## vector

### O(1)删除vector元素

使用move右值转移所有权，将最后一个元素覆盖要删除的元素，
pop_back时间复杂度是O(1)的，由此不必每次删除时移动元素，
时间效率提高，但会破坏原来的数组顺序。

```c++
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;
// 通过下标索引删除元素
template <typename t>
void quick_remove_at(vector<t>& v, int idx) {
    if (idx < v.size()) {
        v[idx] = std::move(v.back());
        v.pop_back();
    }
}

// 通过迭代器删除元素
template <typename t>
void quick_remove_at(vector<t>& v, typename vector<t>::iterator it) {
    if (it != v.end()) {
        *it = std::move(v.back());
        v.pop_back();
    }
}

int main() {
    vector<int> v {0, 1, 2, 3, 4, 5};
    // lambda表达式打印v
    auto print_v ([](vector<int>& v) {
        for (int i : v) {
            cout << i << " ";
        }
        cout << endl;
    });

    print_v(v);
    v.quick_remove_at(v, 3);
    print_v(v);
    v.quick_remove_at(v, find(begin(v), end(v), 1));
    print_v(v);
}
```



# 并行计算

## 多线程

### 同步并行中使用std::cout

设置同步互斥锁即可，可以继承输出流对象，在析构时加锁、打印缓存的字符串

```c++
#include <iostream>
#include <sstream>
#include <mutex>
#include <thread>
#include <vector>

using namespace std;

struct pcout : public: stringstream {
    static inline mutex cout_mutex;
    ~pcout() {
        lock_guard<mutex> l {cout_mutex};
        cout << rdbuf();
    }
};

static void print(int id) {
    pcout{} << "hello world from " << id << " thread\n";
}

int main() {
    vector<thread> v;
    for (int i = 0; i < 10; i++) {
        v.emplace_back(print, i);
    }
    for (auto &t : v) {
        t.join();
    }
}
```

### 单线程生产者&消费者

使用condition_variable条件阻塞，防止消费者太快，申请到锁后
队列却是空的，降低了循环申请锁带来的性能消耗。

```c++
#include <iostream>
#include <mutex>
#include <thread>
#include <queue>
#include <condition_variable>

using namespace std;
mutex mut;
queue<int> q;
bool finishd;
condition_variable cv;

static void producer(int items) {
    for (int i = 0; i < items; i++) {
        this_thread::sleep_for(100ms);
        {
            lock_guard<mutex> lock {mut};
            q.push(i);
            cout << "Producer ---> item " << q.back() << endl;
        }
        cv.notify_all();
    }
    {
        lock_guard<mutex> lock {mut};
        cout << "Producer exit" << endl;
    }
    cv.notify_all();
}

static void consumer() {
    while (!finishd || !q.empty()) {
        unique_lock<mutex> lock {mut};
        cv.wait(lock, [&] { return !q.empty() || finished; });
        if (!q.empty()) {
            cout << "consumer gets itme " << q.front() << endl;
            q.pop();
        }
    }
    cout << "Consumer exit" << endl;
}

int main() {
    thread t1 {producer, 15};
    thread t2 {consumer};

    t1.join();
    t2.join();
}
```

### 多线程消费者&生产者模型

和单线程类似，这里加一个限制队列中item个数不超过stock。

用stopped_produce标志生产者是否停止来决定消费者是否继续循环申请。

另外注意多线程同步互斥打印信息。

```c++
#include <iostream>
#include <mutex>
#include <thread>
#include <vector>
#include <queue>
#include <sstream>
#include <condition_variable>

using namespace std;
mutex mut;
condition_variable go_consume;
condition_variable go_produce;
bool stopped_produce;
queue<int> q;

struct pcout : public stringstream {
    static inline mutex cout_mut;
    ~pcout() {
        lock_guard<mutex> lock {cout_mut};
        cout << rdbuf();
    }
};

void producer(int id, int items, int stock) {
    pcout{} << "P" << id << " start\n";
    for (int i = 1; i <= items; i++) {
        {
            unique_lock<mutex> lock {mut};
            // 限制队列里item的个数不超过stock
            go_produce.wait(lock, [&] { return q.size() < stock; });
            q.push(id * 100 + i);
            pcout{} << "    P" << id << " ---> " << q.back() << endl;
        }
        // 通知消费者消费
        go_consume.notify_all();
        this_thread::sleep_for(130ms);
    }
    pcout{} << "P" << id << " exit\n";
}

void consumer(int id) {
    pcout{} << "C" << id << " start\n";
    while (!stopped_produce) {
        unique_lock<mutex> lock {mut};
        // 设置延时等待不超过1秒
        if (go_consume.wait_for(lock, 1s, [&] { return stopped_produce || !q.empty(); })) {
            pcout{} << "C" << id << " <--- " << q.front() << endl;
            q.pop();
            this_thread::sleep_for(90ms);
            // 通知生产者生产
            go_produce.notify_all();
        }
    }
    pcout{} << "C" << id << " exit\n";
}

int main() {
    vector<thread> workers;
    vector<thread> consumers;
    for (int i = 1; i <= 3; i++)
        workers.emplace_back(producer, i, 15, 8);
    for (int i = 1; i <= 5; i++)
        consumers.emplace_back(consumer, i);

    for (auto &t : workers) t.join();
    stopped_produce = true;
    for (auto &t : consumers) t.join();
    pcout{} << "Main Exit\n";
}

```

# 字符串

## 消除字符串首尾空格

找出首尾位置，截取起始位和长度即可，substr需要传入起始位置，截取长度，拷贝获取子串。
迭代器寻找，未找到则返回string::npos，一个特殊位置。

```c++
#include <string>
#include <iostream>

using namespace std;
string trim_whitespace_surrounding(const string& s) {
    const char whitespace[] {" \t\n"};
    const size_t first (s.find_first_not_of(whitespace));
    if (string::npos == first) {
        return {};
    }
    const size_t last (s.find_last_not_of(whitespace));
    return s.substr(first, last - first + 1);
}

int main() {
    string s {" \n\t string surrounded \n\t"};
    cout << "{" << s << "}" << endl;
    cout << "{" << trim_whitespace_surrounding(s) << "}" << endl;
}
```

## string_view

string_view是对字符串的引用，维护位置和长度，避免多余的拷贝和内存分配，
但去掉了终止符，string_view.data()慎用，小心内存泄露。


无需构造去掉首尾空格
```c++
#include <iostream>
#include <string_view>

using namespace std;
void print(string_view v) {
    const size_t words_begin (v.find_first_not_of(" \t\n"));
    // string_view::npos是无符号数取-1值，是一个很大的值；删除前缀
    v.remove_prefix(min(words_begin, v.size());
    const size_t words_end (v.find_last_not_of(" \t\n"));
    if (words_end != string_view::npos) {
        // 删除后缀
        v.remove_suffix(v.size() - words_end - 1);
    }
    cout << "length: " << v.length() << 
        << "[" << v << "]" << endl;
}
```

注意string_view的字符串拼接时构先构造出string，避免因没有终止符而溢出

```c++
string a {"hello"}
string_view b {" world"};
cout << a + string{b} << endl;
```


# 工具

## 内存管理

### unique_ptr指针

所有权只能让一个对象拥有，不能让多个对象指向同一块动态分配的对象。
重点分析下面代码实例内存的回收时机。

```c++
#include <iostream>
#include <memory>

using namespace std;

class A {
public:
    string name;
    A(string n)
        : name(n)
    {
        cout << "create " << name << endl;
    }
    ~A() {
        cout << "delete " << name << endl;
    }
};

// 这里的p不是引用传参过来的，因此函数结束后即离开作用域，然后被销毁
void process_item(unique_ptr<A> p) {
    if (!p) return;
    cout << "processing " << p->name << endl;
}

int main() {
    // 集中构造unique_ptr的方式，推荐第三种，auto + make_unique
    unique_ptr<A> q1 {new A("item1")};
    unique_ptr<A> q2 {make_unique<A>("item2")};
    auto q3 {make_unique<A>("item3")};

    // unique_ptr不支持拷贝复制，内存是指针独有的，可以用move转移所有权
    // 因此执行完后p2就会被销毁
    process_item(move(p2));
    process_item(make_unique<A>("foo4"));

    cout << "end of main" << endl;
}

```

### shared_ptr

shared_ptr增加了计数来维护指向同一块内存的指针个数，释放时计数减一，
为0时释放内存。重点分析下列代码实例的计数情况。

```c++
#include <iostream>
#include <memory>

using namespace std;

class A {
public:
    string name;
    A(string n) : name(n) { cout << "create " << name << endl; }
    ~A() { cout << "delete " << name << endl; }
};

void print_count(shared_ptr<A>& p) {
    cout << p.use_count() << endl;
}

int main() {
    shared_ptr<A> p1 {make_shared<A>("p1")};
    print_count(p1);
    shared_ptr<A> p2 {p1};
    print_count(p2);
    shared_ptr<A> p3;
    p3 = p2;
    print_count(p1);
}
```

内存被shared_ptr指针拥有时是内存安全的，但要注意以下情况

```c++
void f(shared_ptr<A> a, shared_ptr<B> b, int c);

...

f(new A, new B, other_f());
```

这时会先创建a, b, other_f，然后再将参数传给函数f，而创建顺序是未知的。
如果在a、b申请了空间后other_f抛出了异常，导致参数未能
传递给f的智能指针，由此造成内存泄露。应该采用下列做法。

```c++
shared_ptr<A> pa;
shared_ptr<B> pb;
int c = other_f();
f(pa, pb, c);

```

### weak_ptr

弱指针不会直接操作指向的内存，不影响其生存期，也无法直接访问内部数据，
而是作为辅助，能查看共享指针的计数情况，本身不参与计数。当需要操作
内存数据时，先上锁，然后通过共享指针间接操作数据。

分析以下代码内存释放的时机，可以发现并不会受到弱指针影响。

```c++
#include <iostream>
#include <memory>

using namespace std;
struct A {
    int value;
    A(int i) : value(i) {}
    ~A() { cout << "delete A:" << value << endl; }
};

void weak_ptr_info(const weak_ptr<A> &p) {
    cout << "-----------------" << endl;
    // expired函数判断是否悬空，是则返回true
    cout << p.expired() << endl;
    cout << p.use_count() << endl;
    // p.lock()，上锁并返回一个共享指针，间接访问数据
    if (const shared_ptr<A> sp (p.lock()); sp) {
        cout << sp->value << " " << sp.use_count() << endl;
    } else {
        cout << "null\n";
    }
}

int main() {
    weak_ptr<A> weak_p;
    weak_ptr_info(weak_p);
    {
        shared_ptr<A> shared_p {make_shared<A>(123)};
        weak_p = shared_p;
        weak_ptr_info(weak_p);
    }
    weak_ptr_info(weak_p);
}
```
