#第四章 同步并发操作

**本章主要内容**

>等待一个事件<br>
>带有期望的等待一次性事件<br>
>在限定时间内等待<br>
>使用同步操作简化代码<br>

在上一章中，我们看到各种在线程间保护共享数据的方法。但又是你不仅想要保护数据，还想对单独的线程进行同步。例如，在第一个线程完成前，其可能需要等待另一个线程执行完成。通常情况下，会让线程等待一个特定事件的发生，或者等待某一条件达成(为true)。虽然，这可能定期检查“任务完成”标识，或将类似的东西存到共享数据中，但这与理想情况差的太远了。线程间需要同步操作，像这种常见的情况，C++标准库提供了一些工具可用于同步操作，形式上表现为条件变量(*condition variables*)和期望(*futures*)。

在本章，将讨论如何使用条件变量等待事件，以及介绍期望，和如何使用它简化同步操作。

##4.1 等待一个事件或其他条件

假设你在旅游，而且正在一辆在夜间运行的火车上。在夜间，如何在正确的站点下车呢？一种方法是整完都要醒着，然后注意到了哪一站。这样，你就不会错过你要到达的站点，但是这样做会让你感到很疲倦。另外，你可以看一下时间表，估计一下火车到达目的地的时间，然后在一个稍早的时间点上设置闹铃，然后你就可以安心的睡会了。虽然，这个方法感觉也很不错，也没有错过你要下车的站点，但是当火车晚点的时候，你就要被过早的叫醒了。当然，你闹钟的电池也可能会没电了，导致你睡过站了。这里理想的方式是，无论是早或晚，只要当火车到站的时候，有人或其他东西能把你唤醒，就好了。

这和线程有什么关系呢？好吧，让我们来联系一下。当一个线程等待另一个线程完成任务时，它会有很多选择。第一，它可以持续的检查共享数据的标志(用于做保护工作的互斥量)，直到另一线程完成工作时对这个标志进行重设。从两方面看，这在就是一种浪费：线程消耗宝贵的执行时间持续的检查对应标志，并且当互斥量被等待线程上锁后，其他线程就没有办法对锁进行获取。这样线程就会持续等待，因为以上方式对等待线程限制资源，甚至是在他们完成时阻碍对标识的设置。这种情况类似与，保持清醒状态和列车驾驶员聊了一晚上：驾驶员不得不缓慢驾驶，因为你分散了他的注意力，所以火车需要更长的时间，才能到站。同样的，等待的线程会等待更长的时间，这些线程也在消耗着系统资源。

第二个选择是在等待线程在检查间隙，使用`std::this_thread::sleep_for()`进行周期性的间歇(详见4.3节)：

```c++
bool flag;
std::mutex m;

void wait_for_flag()
{
  std::unique_lock<std::mutex> lk(m);
  while(!flag)
  {
    lk.unlock();  // 1 解锁互斥量
    std::this_thread::sleep_for(std::chrono::milliseconds(100));  // 2 休眠100ms
    lk.lock();   // 3 再锁互斥量
  }
}
```

在这个循环中，在休眠前②，函数对互斥量进行解锁①，并且在休眠结束后再对互斥量进行上锁，所以另外的线程就有机会获取锁并设置标识。

这是一个进步，因为当线程休眠时，线程没有浪费执行时间，但是很难确定正确的休眠时间。太短的休眠和没有休眠一样，都会浪费执行时间；太长的休眠时间，可能会让任务等待线程醒来。休眠时间过长是很少见的情况，因为这会直接影响到程序的行为，当在高节奏游戏(fast-paced game)中，它以为着丢帧，或在一个实时应用中超越了一个时间片。

第三个选择(也是优先的选择)是，使用C++标准库提供的工具去等待事件的发生。通过另一线程触发等待事件的机制，是最基本的唤醒方式(例如：如流水线上存在额外的任务时)，并且这种机制就称为“条件变量”(condition variable)。从概念上来说，一个条件变量会与多个事件或其他条件相关，并且一个或多个线程会等待条件的达成。当某些线程被终止时，为了将等待线程唤醒，并且允许他们继续执行，终止的线程将会向等待着的线程广播“条件达成”的信息。

###4.1.1 等待条件达成

C++标准库对条件变量有两套实现：`std::condition_variable`和`std::condition_variable_any`。都包含在<condition_variable>头文件中。两种类型都需要与一个互斥量一起才能工作，互斥量在这里可进行适当的同步；前者会被`std::mutex`限制，而后者和任何满足最低标准的互斥量一起时，可自由的工作，为了强调“任何”，从而加上了*_any*的后缀。因为` std::condition_variable_any`更加通用，这就可能从体积、性能，以及系统资源的使用方面产生额外的开销，所以`std::condition_variable`一般作为首选的类型，当对灵活性有硬性要求时，我们才会去考虑`std::condition_variable_any`。

所以，如何使用`std::condition_variable`去处理之前提到的情况——当有数据需要处理时，如何唤醒休眠中的线程对其进行处理？以下清单展示了一种使用条件变量做唤醒的方式。

清单4.1 使用`std::condition_variable`处理数据等待
```c++
std::mutex mut;
std::queue<data_chunk> data_queue;  // 1
std::condition_variable data_cond;

void data_preparation_thread()
{
  while(more_data_to_prepare())
  {
    data_chunk const data=prepare_data();
    std::lock_guard<std::mutex> lk(mut);
    data_queue.push(data);  // 2
    data_cond.notify_one();  // 3
  }
}

void data_processing_thread()
{
  while(true)
  {
    std::unique_lock<std::mutex> lk(mut);  // 4
    data_cond.wait(
         lk,[]{return !data_queue.empty();});  // 5
    data_chunk data=data_queue.front();
    data_queue.pop();
    lk.unlock();  // 6
    process(data);
    if(is_last_chunk(data))
      break;
  }
}
```

首先，这里传递一些数据到队列①中供两个线程使用。当数据准备好时，队列使用`std::lock_guard`，并将准备好的数据压入队列中②，之后线程会对队列中的数据上锁。然后调用`std::condition_variable`的notify_one()成员函数，对等待的线程(如果有等待线程)进行通知③。

另一种情况，当你有一个正在处理数据的线程。这个线程首先对互斥量上锁，但在这里`std::unique_lock`要比`std::lock_guard`④更加合适——且听我细细道来。线程之后会调用`std::condition_variable`的成员函数wait()，传递一个锁和一个lambda函数表达式(作为等待的条件⑤)。Lambda函数是C++11添加的行特性，它可以让一个匿名函数作为其他表达式的一部分，并且非常合适作为标准函数的谓词，例如wait()函数。在这个例子中，简单的lambda函数**[]{return !data_queue.empty();}**会去检查data_queue，当data_queue不为空时——那就意味着队列中已经有准备好数据了。附录A的A.5节有Lambda函数更多的信息。

wait()会去检查这些条件(通过调用给定的lambda函数)，当条件满足(lambda函数返回true)时返回。当不满足条件(lambda函数返回false)，wait()函数将解锁互斥量，并且将这个线程(上段提到的处理数据的线程)置于阻塞或等待状态。当条件变量满足时，准备数据线程就会调用notify_one()，这个线程将会被唤醒，去获取互斥锁，并且对条件再次检查，因为wait()可能会在条件满足的情况下，继续持有锁。当条件不满足时，线程将对互斥量解锁，并且重新开始等待。这就是为什么用`std::unique_lock`而不使用`std::lock_guard`——等待中的线程必须在解锁互斥量的同时，等待互斥锁再次上锁，`std::lock_guard`没有这么灵活。当这个线程休眠时，互斥量保持锁住状态。准备数据的线程不会锁住互斥量，用来添加任务到队列中；同样的，等待线程也永不会知道条件何时达成。

清单4.1使用了一个简单的lambda函数用于等待⑤，这个函数用于检查队列何时不为空，不过任意的函数和可调用对象都可以传入wait()。当你已经写好了一个函数去做检查条件(或许比清单中简单检查要复杂很多)，那就可以直接将这个函数传入wait()；不一定非要放在一个lambda表达式中。在调用wait()的过程中，一个条件变量可能会去检查给定条件若干次；不过，这种事情也发生在互斥量上，这样(仅)当提供测试条件的函数返回true时，结果就能被理解返回了。当等待线程重新获取互斥量，并且检查条件时，当另一个线程没有在第一时间相应，这就是所谓的“虚假唤醒”(*spurious wakeup*)。因为任何虚假唤醒的数量和频率都是不确定的，这里不建议使用一个有副作用的函数做条件检查。当你这样做了，就要有这种副作用将会接二连三的产生的心理准备。

灵活的解锁`std::unique_lock`，不仅适用于去调用wait()；当你有数据需要处理，单还没有处理时⑥，你也会用到它。处理数据可能是一个耗时的操作，并且当你看过第3章后，你i就知道持有锁的时间过长是一个多么糟糕的主意。

使用队列在多个线程中转移数据(如清单4.1)很常见。做的好的话，同步操作会限制队列本身，同步问题和条件竞争出现的概率也会降低。鉴于这些好处，现在从清单4.1中提取出一个通用线程安全的队列。

###4.1.2 使用条件变量构建线程安全队列

当你正在设计一个通用队列时，花一些时间想想有哪些操作需要添加到队列实现中去，就如之前在3.2.3节看到的线程安全的栈。看一下C++标准库提供的实现，找找灵感；`std::queue<>`容器的接口展示如下：

清单4.2 `std::queue`接口
```c++
template <class T, class Container = std::deque<T> >
class queue {
public:
  explicit queue(const Container&);
  explicit queue(Container&& = Container());
  template <class Alloc> explicit queue(const Alloc&);
  template <class Alloc> queue(const Container&, const Alloc&);
  template <class Alloc> queue(Container&&, const Alloc&);
  template <class Alloc> queue(queue&&, const Alloc&);

  void swap(queue& q);

  bool empty() const;
  size_type size() const;

  T& front();
  const T& front() const;
  T& back();
  const T& back() const;

  void push(const T& x);
  void push(T&& x);
  void pop();
  template <class... Args> void emplace(Args&&... args);
};
```

当你忽略构造、赋值以及交换操作时，你就剩下了三组操作：1. 对整个队列的状态进行查询(empty()和size());2.查询在队列中的各个元素(front()和back())；3.修改队列的操作(push(), pop()和emplace())。这就和3.2.3中的栈一样了，因此你也会遇到在固有接口上的条件竞争。因此，你需要将front()和pop()合并成一个函数调用，就像之前在栈实现时合并top()和pop()一样。与清单4.1中的代码不同的是：当使用队列在多个线程中传递数据时，接收线程进程需要等待数据的压入。这里我们提供pop()函数的两个变种：try_pop()和wait_and_pop()。try_pop() ，尝试从队列中弹出数据，但总直接返回(当有失败时)，即使没有指可检索；wait_and_pop()，将会等待有值可检索的时候才返回。当你使用之前栈的方式来实现你的队列，你实现的队列接口就可能会下面这副摸样：

清单4.3 线程安全队列的接口
```c++
#include <memory> // 为了使用std::shared_ptr

template<typename T>
class threadsafe_queue
{
public:
  threadsafe_queue();
  threadsafe_queue(const threadsafe_queue&);
  threadsafe_queue& operator=(
      const threadsafe_queue&) = delete;  // 不允许简单的赋值

  void push(T new_value);

  bool try_pop(T& value);  // 1
  std::shared_ptr<T> try_pop();  // 2

  void wait_and_pop(T& value);
  std::shared_ptr<T> wait_and_pop();

  bool empty() const;
};
```

就像之前对栈做的那样，在这里你将很多构造函数剪裁掉了，并且禁止了对队列的简单赋值。和之前一样，你也需要提供两个版本的try_pop()和wait_for_pop()。第一个重载的try_pop()①在引用变量中存储着检索值，所以它可以用来返回值的状态；当检索到一个变量时，他将返回true，否则将返回false(详见A.2节)。第二个重载②就不能做这事了，以为它是用来直接返回检索值的。但当没有值可检索时，这个函数也可以返回NULL指针。

那么问题来了，如何将以上这些和清单4.1中的代码相关联呢？好吧，我们现在就来看看怎么去关联，你可以从之前的代码中提取push()和wait_and_pop()，如以下清单所示。

清单4.4 从清单4.1中提取push()和wait_and_pop()
```c++
#include <queue>
#include <mutex>
#include <condition_variable>

template<typename T>
class threadsafe_queue
{
private:
  std::mutex mut;
  std::queue<T> data_queue;
  std::condition_variable data_cond;
public:
  void push(T new_value)
  {
    std::lock_guard<std::mutex> lk(mut);
    data_queue.push(new_value);
    data_cond.notify_one();
  }

  void wait_and_pop(T& value)
  {
    std::unique_lock<std::mutex> lk(mut);
    data_cond.wait(lk,[this]{return !data_queue.empty();});
    value=data_queue.front();
    data_queue.pop();
  }
};
threadsafe_queue<data_chunk> data_queue;  // 1

void data_preparation_thread()
{
  while(more_data_to_prepare())
  {
    data_chunk const data=prepare_data();
    data_queue.push(data);  // 2
  }
}

void data_processing_thread()
{
  while(true)
  {
    data_chunk data;
    data_queue.wait_and_pop(data);  // 3
    process(data);
    if(is_last_chunk(data))
      break;
  }
}
```

线程队列的实例现在包含有互斥量和条件变量，所以独立的变量就不再需要了①，并且调用push()也不需要外部同步②。wait_and_pop()还要兼顾条件变量的等待③。

另一个wait_and_pop()函数的重载写起来就很琐碎了，剩下的函数就像从清单3.5实现的栈中一个个的粘过来一样。最终的队列实现如下所示。

清单4.5 使用条件变量的线程安全队列(完整版)
```c++
#include <queue>
#include <memory>
#include <mutex>
#include <condition_variable>

template<typename T>
class threadsafe_queue
{
private:
  mutable std::mutex mut;  // 1 互斥量必须是可变的 
  std::queue<T> data_queue;
  std::condition_variable data_cond;
public:
  threadsafe_queue()
  {}
  threadsafe_queue(threadsafe_queue const& other)
  {
    std::lock_guard<std::mutex> lk(other.mut);
    data_queue=other.data_queue;
  }

  void push(T new_value)
  {
    std::lock_guard<std::mutex> lk(mut);
    data_queue.push(new_value);
    data_cond.notify_one();
  }

  void wait_and_pop(T& value)
  {
    std::unique_lock<std::mutex> lk(mut);
    data_cond.wait(lk,[this]{return !data_queue.empty();});
    value=data_queue.front();
    data_queue.pop();
  }

  std::shared_ptr<T> wait_and_pop()
  {
    std::unique_lock<std::mutex> lk(mut);
    data_cond.wait(lk,[this]{return !data_queue.empty();});
    std::shared_ptr<T> res(std::make_shared<T>(data_queue.front()));
    data_queue.pop();
    return res;
  }

  bool try_pop(T& value)
  {
    std::lock_guard<std::mutex> lk(mut);
    if(data_queue.empty())
      return false;
    value=data_queue.front();
    data_queue.pop();
    return true;
  }

  std::shared_ptr<T> try_pop()
  {
    std::lock_guard<std::mutex> lk(mut);
    if(data_queue.empty())
      return std::shared_ptr<T>();
    std::shared_ptr<T> res(std::make_shared<T>(data_queue.front()));
    data_queue.pop();
    return res;
  }

  bool empty() const
  {
    std::lock_guard<std::mutex> lk(mut);
    return data_queue.empty();
  }
};
```

尽管empty()是一个const成员函数，并且传入拷贝构造函数的other形参是一个const引用；因为其他线程可能有这个类型的非const引用对象，并调用变种成员函数，所以这里有必要对互斥量上锁。如果锁住互斥量是一个可变操作，那么这个互斥量对象就会标记为可变的①，之后他就可以在empty()和拷贝构造函数中被锁住了。

条件变量在多个线程等待同一个事件时，也是很有用的。当线程用来分解工作负载，并且只有一个线程可以对通知做出反应，与清单4.1中使用的结构完全相同；至运行多个数据实例——处理线程(*processing thread*)。当新的数据准备完成，对notify_one()的调用将会触发一个正在执行wait()的线程，去检查条件和wait()函数的返回状态(因为你仅是向data_queue添加一个数据项)。 这里不保证线程一定会被通知到，即使只有一个等待线程被通知；所有处线程可能都在处理数据。

另一种可能是，很多线程等待同一个事件，对于通知他们都需要做出回应。这会发生在共享数据正在被初始化的时候，在处理线程可以使用同一数据时，就要等待数据被初始化(有不错的机制可用来应对；可见第3章，3.3.1节)，或等待共享数据的更新，比如定期的重新初始化(*periodic reinitialization*)。在这些情况下，准备线程准备数据数据时，就会通过条件变量调用notify_all()成员函数，而非直接调用notify_one()函数。顾名思义，这就是全部线程在都去执行wait()(检查他们等待的条件是否满足)的原因。

当等待线程只会等待一次，所以当条件为true时，它就不会再等待条件变量了，所以一个条件变量可能并非同步机制的最好选择。尤其是，条件在等待一组可用的数据块时。在这样的情况下，期望(*future*)就是一个适合的选择。

##4.2 使用期望等待一次性事件

假设你乘飞机去国外度假。当你到达机场，并且办理完各种登机手续后，你还需要等待机场广播通知你登机，可能要等很多个小时。你可能会在候机室里面找一些事情来打发时间，比如：读书，上网，或者来一杯价格不菲的机场咖啡，不过从根本上来说你就在等待一件事情：机场广播能够登机的时间。给定的飞机班次再之后没有可参考性；当你在再次度假的时候，你可能会等待另一班飞机。

C++标准库模型将这种一次性事件称为“期望” (*future*)。当一个线程需要等待一个特定的一次性事件，在某种程度上来说它就需要这个事件在未来的表现形式。之后，这个线程会周期性(较短的周期)的等待或检查，事件是否触发(检查信息板)；在检查期间也会执行其他任务(品尝昂贵的咖啡)。另外，在等待任务期间它可以先执行另外一个任务，直到对应的任务触发，而后等待期望的状态会变为“就绪”(*ready*)。一个期待可能是数据相关的(比如，你的登机口编号)，也可能不是。当事件发生时(并且期望状态为就绪)，这个期望就不能被重置。

在C++标准库中，有两种期望，使用两种类型模板实现，并且声明在<future>头文件中: 唯一期望(*unique futures*)(`std::future<>`)和共享期望(*shared futures*)(`std::shared_future<>`)。这是仿照`std::unique_ptr`和`std::shared_ptr`。`std::future`的实例只能与一个指定事件相关联，而`std::shared_future`的实例就能关联多个事件。后者的实现中，所有实例会在同时变为就绪状态，并且他们可以访问与事件相关的任何数据。这种数据关联与模板有关；比如`std::unique_ptr` 和`std::shared_ptr`的模板参数就是相关联的数据类型。在与数据无关的地方，可以使用`std::future<void>`与`std::shared_future<void>`的特化模板。虽然，我期望用于线程间的通讯，但是期望对象本身并不提供同步访问。当多个线程需要访问一个独立期望对象时，他们必须使用互斥量或这其他同步机制对访问进行保护，如在第3章提到的。不过，在你将要阅读到的4.2.5节，多个线程会访问他们对一个`std::shared_future<>`实例的副本，而不需要期望同步，即使他们是同意个异步结果。

最基本的一次性事件，就是一个后台运行出的计算结果。在第2章中，你已经了解了`std::thread` 执行的任务不能有返回值，并且我能保证，这个问题将在使用期望后解决——现在就来看看是怎么解决的。

###4.2.1 带返回值的后台任务

假设，你有一个需要长时间的运算，你需要其最终能计算出一个有效的值，但是你现在并不需要这个值。可能你已经找到了生命、宇宙，以及万物的答案，就像道格拉斯·亚当斯[1]一样。你可以启动一个新线程来执行这个计算，但是这就以为这你必须关注如何传回计算的结果，因为`std::thread`并不提供直接接收返回值的机制。这里就需要`std::async`函数模板(也是在头文件<future>中声明的)了。

你可以使用`std::async`启动一个异步任务，任务的结果你不着急要。与让`std::thread`对象等待运行方式的不同，`std::async`会返回一个`std::future`对象，这个对象持有最终计算出来的结果。当你需要这个值时，你只需要调用这个对象的get()成员函数；并且直到期望状态为就绪的情况下，线程才会阻塞；之后，返回计算结果。下面清单中代码就是一个简单的例子。

清单4.6 使用`std::future`从异步任务中获取返回值
```c++
#include <future>
#include <iostream>

int find_the_answer_to_ltuae();
void do_other_stuff();
int main()
{
  std::future<int> the_answer=std::async(find_the_answer_to_ltuae);
  do_other_stuff();
  std::cout<<"The answer is "<<the_answer.get()<<std::endl;
}
```

`std::async`允许你通过添加额外的调用参数，向函数传递额外的参数，与`std::thread` 做的方式一样。当第一个参数是一个指向成员函数的指针，第二个参数提供有这个函数成员类的具体对象(不是直接的，就是通过指针，还可以包装在`std::ref`中)，剩余的参数可作为成员函数的参数传入。否则，第二个和随后的参数将作为函数的参数，或作为指定可调用对象的第一个参数。就如`std::thread`，当参数右值(*rvalues*)时，拷贝操作将使用移动的方式转移原始数据。这就允许使用“只移动”类型作为函数对象和参数。来看一下下面的程序清单：

 清单4.7 使用`std::async`向函数传递参数
```c++
#include <string>
#include <future>
struct X
{
  void foo(int,std::string const&);
  std::string bar(std::string const&);
};
X x;
auto f1=std::async(&X::foo,&x,42,"hello");  // 调用p->foo(42, "hello")，p是指向x的指针
auto f2=std::async(&X::bar,x,"goodbye");  // 调用tmpx.bar("goodbye")， tmpx是x的拷贝副本
struct Y
{
  double operator()(double);
};
Y y;
auto f3=std::async(Y(),3.141);  // 调用tmpy(3.141)，tmpy通过Y的移动构造函数得到
auto f4=std::async(std::ref(y),2.718);  // 调用y(2.718)
X baz(X&);
std::async(baz,std::ref(x));  // 调用baz(x)
class move_only
{
public:
  move_only();
  move_only(move_only&&)
  move_only(move_only const&) = delete;
  move_only& operator=(move_only&&);
  move_only& operator=(move_only const&) = delete;
  
  void operator()();
};
auto f5=std::async(move_only());  // 调用tmp()，tmp是通过std::move(move_only())构造得到
```

在默认情况下，这取决于`std::async`是否启动一个线程，或是否在期望等待时同步任务。在大多数情况下，估计这就是你想要的结果，但是你也可以在函数调用之前，向`std::async`传递一个额外参数。这个参数的类型是`std::launch`，还可以是`std::launch::defered`，用来表明函数调用被延迟到wait()或get()函数调用时才执行，`std::launch::async` 为了表明函数必须在其所在的独立线程上执行，`std::launch::deferred | std::launch::async`为了表明实现可以选择这两种方式的一种。最后一个选项是默认的。当函数调用被延迟，它可能不会在运行了。如下所示：

```c++
auto f6=std::async(std::launch::async,Y(),1.2);  // 在新线程上执行
auto f7=std::async(std::launch::deferred,baz,std::ref(x));  // 在wait()或get()调用时执行
auto f8=std::async(
              std::launch::deferred | std::launch::async,
              baz,std::ref(x));  // 实现选择执行方式
auto f9=std::async(baz,std::ref(x));
f7.wait();  //  调用延迟函数
```

在本章的后面和第8章中，你将会再次看到这段程序，使用`std::async`会让分割算法到各个任务中变的容易，这样程序就能并发的执行了。不过，这不是让一个`std::future`与一个任务实例相关联的唯一方式；你也可以将任务包装入一个`std::packaged_task<>`实例中，或通过编写代码的方式，使用`std::promise<>`类型模板显示设置值。与`std::promise<>`对比，`std::packaged_task<>`具有更高层的抽象，所以我们从“高抽象”的模板说起。

###4.2.2 任务与期望

`std::packaged_task<>`对一个函数或可调用对象，绑定一个期望。当`std::packaged_task<>` 对象被调用，它就会调用相关函数或可调用对象，将期望状态置为就绪，返回值也会被存储为相关数据。这可以用在构建线程池的建筑块(可见第9章)，或用于其他任务的管理，比如在任务所在线程上运行任务，或将它们顺序的运行在一个特殊的后台线程上。当一个粒度较大的操作可以被分解为独立的子任务时，其中每个子任务就可以包含在一个`std::packaged_task<>`实例中，之后这个实例将传递到任务调度器或线程池中。这就是对任务的细节进行抽象了；调度器仅处理`std::packaged_task<>`实例，要比处理独立的函数高效的多。

***
[1] In *The Hitchhiker’s Guide* to the Galaxy, the computer Deep Thought is built to determine “the answer to Life,
the Universe and Everything.” The answer is 42



