主要看的是智能指针与移动语义。参考的是CMU 15-445， 2022 fall

参考：

https://blog.csdn.net/Xiejingfa/article/details/50759210

-   `unique_ptr` 把握unique含义，资源的所有权不是共享的，而是被某个对象独占的时候。当资源所有权发生转移，可以通过move或者release进行转移
-   `shared_ptr` 把握share的含义，对象的所有权是**共享**的，可以被拷贝

## unique_ptr
### unique_ptr教程
`unique_ptr`的创建：

- 创建空的`unique_ptr`
	- `std::unique_ptr<int> p1();`
	- `std::unique_ptr<int> p2(nullptr);`
- 创建有明确指向的内存地址的`unique_ptr`
	- `std::unique_ptr<int> p3(new int);`
- 利用移动语义创建
```c++
std::unique_ptr<int> p4(new int);
std::unique_ptr<int> p5(p4);//错误，堆内存不共享
std::unique_ptr<int> p5(std::move(p4));//正确，调用移动构造函数
```
值得一提的是，对于调用移动构造函数的 p4 和 p5 来说，p5 将获取 p4 所指堆空间的所有权，而 p4 将变成空指针（nullptr）。
- 使用make创建
	- `p1 = make_unique<string>(10, '9');`（括号里写的是构造函数的参数，优先这种）

`unique_ptr`包含的成员方法：
- `get()`：获取当前 unique_ptr 指针内部包含的普通指针。
- `release()`：释放当前 unique_ptr 指针对所指堆内存的所有权，但该存储空间并不会被销毁。

### Bustub中的`unique_ptr`
```c++
class TrieNode {
 public:
  explicit TrieNode(char key_char) {}
  TrieNode(TrieNode &&other_trie_node) noexcept {}
  virtual ~TrieNode() = default;
  bool HasChild(char key_char) const { return false; }
  bool HasChildren() const { return false; }
  bool IsEndNode() const { return false; }
  char GetKeyChar() const { return 'a'; }
  std::unique_ptr<TrieNode> *InsertChildNode(char key_char, std::unique_ptr<TrieNode> &&child) { return nullptr; }
  std::unique_ptr<TrieNode> *GetChildNode(char key_char) { return nullptr; }
  void RemoveChildNode(char key_char) {}
  void SetEndNode(bool is_end) {}

 protected:
  char key_char_;
  bool is_end_{false};
  std::unordered_map<char, std::unique_ptr<TrieNode>> children_;
};
```

我们能够看到，这里出现了很多`unique_ptr`这一种智能指针。

出现`unique_ptr`，就意味这个对象的生命周期由它来管理！也就是说，所有的`TrieNode`的生命周期都是由成员变量`children_`来管理的。那其他部分需要使用这个`TrieNode`的话，可以使用**裸**指针或者`unique_ptr`的**指针**来用。

**什么时候用`unique_ptr`？**
对于一个对象A，存在另一个对象B，B的生命周期>=A，那么就可以由B来控制A的生命周期，即拥有对象A的`unique_ptr`（往往是作为成员变量），而其他对象使用对象A时，只能用对象A的裸指针或者`unique_ptr`的指针，且需要保证在使用时，对象A没有被释放。

所以，`TrieNode`节点对它的子`TrieNode`节点的生命周期拥有管理权，故拥有它的`unique_ptr`，而其他节点如果想要通过`TrieNode`节点去访问其子`TrieNode`节点的话，只能使用子`TrieNode`的裸指针或者`unique_ptr`的指针。


**工厂模式中的`unique_ptr`**
unique_ptr不支持拷贝操作，但却有一个例外：可以从函数中返回一个unique_ptr。
示例：
```C++
unique_ptr<int> clone(int p)
{
    unique_ptr<int> pInt(new int(p));
    return pInt;    // 返回unique_ptr
}
int main() {
    int p = 5;
    unique_ptr<int> ret = clone(p);
    cout << *ret << endl;
}
```
也是由于这个原因，我们可以看到Bustub中很多函数的**返回值**就是一个`unique_ptr`智能指针，注意哦，`TrieNode`里返回的是`unique_ptr`智能指针的指针。
例如`ExecutorFactory::CreateExecutor()`：
```c++
auto ExecutorFactory::CreateExecutor(ExecutorContext *exec_ctx, const AbstractPlanNodeRef &plan)
    -> std::unique_ptr<AbstractExecutor> {
  switch (plan->GetType()) {
    // Create a new sequential scan executor
    case PlanType::SeqScan: {
      return std::make_unique<SeqScanExecutor>(exec_ctx, dynamic_cast<const SeqScanPlanNode *>(plan.get()));
    }
    // Create a new index scan executor
    case PlanType::IndexScan: {
      return std::make_unique<IndexScanExecutor>(exec_ctx, dynamic_cast<const IndexScanPlanNode *>(plan.get()));
    }
    //.....
```

这里面就直接`return std::make_unique<>`了，在外面使用如下代码接受：
```c++
auto child_executor = ExecutorFactory::CreateExecutor(exec_ctx, update_plan->GetChildPlan());
```
此时，`child_executor`就是返回的智能指针了。

综上所述，`unique_ptr`智能指针可以出现在函数的返回值中：
- 如果是裸指针或者`unique_ptr`的指针，那么意味着所有权**没有发生转移**
- 如果是`unique_ptr`，那么意味着所有权**发生了转移**


除了出现在成员变量、函数返回值之外，`unique_ptr`还出现在函数参数中。
以`InsertChildNode`为例：
```c++
  std::unique_ptr<TrieNode> *InsertChildNode(char key_char, std::unique_ptr<TrieNode> &&child) {
    if (HasChild(key_char) || child->GetKeyChar() != key_char) {
      return nullptr;
    }
    // 右值引用还是左值，所以需要使用std::move将其转化为右值，这样就会调用map的移动赋值
    children_[key_char] = std::move(child);
    // 这个只是返回unique_ptr智能指针的指针，所以直接传即可。
    return &children_[key_char];
  }
```
在这里出现了`TrieNode &&other_trie_node`这个用法，这是`unique_ptr`智能指针的右值引用。

注意，右值引用只是一种**类型强转**，并没有做任何**资源窃取**等能加速的操作，真正做了**资源窃取**的操作是落在了下面这一行上：
```c++
    children_[key_char] = std::move(child);
```
在这里，调用了`unique_ptr`的移动构造函数，而`unique_ptr`的移动构造函数是默认做好了的，在这个移动构造函数里面，实现了资源的**窃取**（也就是把传入参数的值给了自己，把传入参数的值置为了null）。
这里为什么还需要用`std::move`呢？
**因为右值引用是左值**。
也就是说，此时`child`**是右值引用，但是它是左值**。
如果我们还需要更进一步利用移动语义的话，我们还需要将其转化为右值，也就是调用`std::move`。也就是在这句代码之后，我们也不会再使用`child`这个传入参数了。
![](../../img/Pasted%20image%2020221209185526.png)

通常的使用习惯：
- 有一个真正有效的移动构造函数、移动赋值函数或者自定义的含有移动入参的函数。
- 真正有效是指，会对右值的所拥有的资源进行窃取，并置右值引用拥有资源为空。
- 层层调用，利用`std::move`将右值一直传递到会进行**资源窃取**的函数（往往是`vector，unique_ptr`等类型默认的移动构造或移动赋值函数）中



综上，`unique_ptr`在bustub中出现在下列场所：
- 类的成员变量
- 函数返回值
- 函数参数

或者以是否需要转移所有权作为划分标准：
- 需要转移所有权：
	- 类的成员变量：`std::unique_ptr`的形式
	- 函数返回值：直接返回`std::unique_ptr`的形式（不具有拷贝函数，这是例外）
	- 函数参数：`std::move()`与`std::unique_ptr&&`
- 不需要转移所有权：
	- 类的成员变量：裸指针 或 `unique_ptr *`
	- 函数返回值：裸指针 或 `unique_ptr *`
	- 函数参数：裸指针 或 `unique_ptr *`


## shared_ptr
### shared_ptr教程

 **使用场景**
1.  shared_ptr 通常使用在共享权不明的场景。有可能多个对象同时管理同一个内存时。
2.  对象的延迟销毁。陈硕在《Linux 多线程服务器端编程》中提到，当一个对象的析构非常耗时，甚至影响到了关键线程的速度。可以使用 `BlockingQueue<std::shared_ptr<void>>` 将对象转移到另外一个线程中释放，从而解放关键线程。

