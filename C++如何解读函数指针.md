今天跟一个朋友聊到函数指针的问题，发现其实很多人不懂得如何解读一个函数指针的声明。其实有一个非常好用的法则，但我观察到很多视频或者文章里并没有提到，只是单纯告诉你他举的例子如何解读，换一个例子又不知道了。我给我那朋友解释完，他直惊呼“卧槽”，所以想分享出来，希望可以帮助到那个同样困惑的你。

首先从简单的例子讲起，你至少需要知道这是一个函数指针：
```cpp
void (*fp)(int);
```

整个声明有三个部分，fp是我们给它的名字，左边那个“*”说明它是一个指针，右边(int)是接受的参数类型，前面void是返回类型。你肯定想，


但是这个呢？
```cpp
void* (*(*fp)(int))[10];
```

很多人即使大概能看懂，也没法自信的说就是这样，更没法给别人解释了。其实知道规则以后，会非常易读。我管这个规则叫右摇左摆法则，非常滑稽的名字，bear with me，但实际使用就是从中间看起，然后看它的右边，再看它的左边，再看右边，再看左边，直到结束。

所以上面这个例子，我们从fp也就是中间开始看，

- 看中间，fp，fp是一个什么呢？
- 看右边，右边什么也没有，遇到括号表示告一段落。
- 看左边，是一个*，也就是pointer，所以fp1是一个指针，是什么样的指针呢？
- 看右边，(int)，是一个接受一个int为参数的函数指针。
- 看左边，是一个*，也就是返回值是一个指针，是一个什么样的指针呢？
- 看右边，是[10]，也就是返回一个指针指向一个数组，这个数组包含10个，10个什么呢？
- 看左边，是void*，哦，原来是包含10个void*的数组。

that's it! 用英文更好理解，因为英文的从句可以让上面这么多步连成一整句话。

(middle) fp1 is → (right, nothing) → (left) a pointer to → (right) a function that takes an int → (left) and returns a pointer to → (right) an array of 10 → (left) void pointers.

连起来就是：fp1 is a pointer to a function that takes an int and returns a pointer to an array of 10 void pointers.

是不是觉得so easy？这个方法我其实是从Thinking in C++, Volume 1学到的，然后自己总结了下，并取了个沙雕但是个人认为有助于理解的名字，不懂为什么其他教程都没有这样教，再来一个：
```cpp
int* (*fp)(int*, int* (*)(int*, int*));
```

fp is a pointer to a function, that takes in an int* and a function pointer, which takes in two int* and returns an int*, and returns an int*.

用中文就是fp是一个函数指针，接受一个int\*和另一个函数指针作为参数，返回一个int\*，它接受的那个函数指针参数呢，是一个接受两个int\*作为参数，返回一个int\*的函数指针。这个稍微tricky的地方是你得能识别出它接受的第二个参数是一个函数指针，它没有名字，因为这是声明类型而已，并没有涉及调用，不需要给它命名，就如同这里的三个例子都没有给其他类型的参数名字是一回事。

如果可以帮助理解，可以这么想象函数指针：
```cpp
void (*)(int) fp;
// fp前面整个部分相当于一个type
// 就如同
```

说实话，我觉得解读函数指针的问题已经水落石出了，没法解释的更清楚了，再复杂也能按照这个逻辑去理解。当然，还有什么问题欢迎留言。第一次写B站专栏，体验不是很好:(，希望能早日支持Markdown。
