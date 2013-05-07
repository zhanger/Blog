# 某人的面试题解答

一些比较弱智的 (例如 `-viewDidLoad` 为什么放上面) 和过于笼统的 (例如 Grand Central Dispatch 的概念) 就不说了.

我写这篇文章是给刚入门还没有什么实际经验的新手看的, 不是让某些骄傲自大眼高手低什么都不懂的傻逼可以背下来然后在面试时蒙混过关. 事实上, 如果这篇文章里的内容你没有 100% 了解, 你根本不应该去找工作, 那纯属是浪费 HR 的时间和你自己的时间.


## copy 和 retain 的区别

copy 是重新分配一块内存将相同的内容复制进去, retain 指向同一块内存而仅增加 retain count.


## getter 和 setter

getter 和 setter 一般由 `@synthesize` 生成. 以这样一个 instance 为例:

```
@property (nonatomic, retain) NSString *myString;
```

### 历史

#### 最初的版本

最初 `@synthesize` 通常这样使用:

```
@synthesize myString;
```

生成的 getter 和 setter 是这样的:

```
// Getter
- (NSString *)myString
{
    return myString;
}

// Setter
- (void)setMyString:(NSString *)aMyString
{
    if (myString != aMyString) {
        [myString release];
        myString = [aMyString retain];
    }
}
```

#### 演进

这样虽然简单, 但随之而来有一个问题, 就是 `myString` 没有任何标记很容易和一些局部的 instances 弄混. 于是发展出来了一些变化:

```
@synthesize myString = _myString;
```

生成的 getter 和 setter 变成了这样:

```
// Getter
- (NSString *)myString
{
    return _myString;
}

// Setter
- (void)setMyString:(NSString *)myString
{
    if (_myString != myString) {
        [_myString release];
        _myString = [myString retain];
    }
}
```

有了 `_` 作为前缀就不容易误把它当做局部的 instance 了.

#### 简化

后来大家都觉得一遍又一遍写 `@synthesize` 很烦, 终于有一天 Apple 自己的员工也受不了了, 于是对新的 compiler 做出了一些改进: 如果一个 `@property` 没有对应的 `@synthesize` (或 `@dynamic` 等) 就会在 compile 时自动添加 `@synthesize myInstance = _myInstance;` 的规则.

### atomic property 的区别

被标记为 atomic 的 property 会在 getter 和 setter 中使用 `@synchronized (self) {...}` 来避免多线程冲突.

### 使用

`_myInstance` (或最初没有前缀的 `myInstance`) 和 `self.myInstance` 是不同的. 平时通常不会在 getter 与 setter 外使用 `_myInstance`, 而是通过 getter 和 setter 来读写.

#### 作为 getter

当 `self.myInstance` 在等式右侧或作为一个 method 的参数等情况时作为 getter 使用.

#### 作为 setter

作为 setter 有两种用法, 一种是 `[self setMyInstance:someInstance];`, 另一种是 `self.myInstance = someInstance;`. 两者是等价的.

### 为什么使用 getter 和 setter

getter 和 setter 的实质是将一个 property 的 getting 和 setting 行为 encapsulate, 我所知道的一些使用他们的原因:

* 方便 debug 时截取.
* 方便添加更多的功能 (例如 property 为 `nil` 时自动 initialize).
* 将内存管理与线程管理包在里面简化逻辑.
* 可以在 subclass 中添加逻辑.
* 更灵活地控制访问权限.


## Category

Category 是为了扩展原有的某个类的功能而不重写或继承那个类时使用的方法.
以我的开源项目 [RFoundation](https://github.com/AlexRezit/RFoundation) 为例, 我想为 NSString 添加 URL 编码转换的功能, 就可以用这种方式实现 (仅截取了部分代码):

```
//
//  NSString+URLCoding.h
//
//  Created by Alex Rezit on 22/11/2012.
//  Copyright (c) 2012 Seymour Dev. All rights reserved.
//

#import <Foundation/Foundation.h>

@interface NSString (URLCoding)

- (NSString *)URLEncodedString;
- (NSString *)URLDecodedString;

@end
```

```
//
//  NSString+URLCoding.m
//
//  Created by Alex Rezit on 22/11/2012.
//  Copyright (c) 2012 Seymour Dev. All rights reserved.
//

#import "NSString+URLCoding.h"

@implementation NSString (URLCoding)

- (NSString *)URLEncodedString
{
    CFStringRef resultString = CFURLCreateStringByAddingPercentEscapes(kCFAllocatorDefault,
                                                                       (CFStringRef)self,
                                                                       NULL,
                                                                       CFSTR("!*'();:@&=+$,/?%#[]"),
                                                                       kCFStringEncodingUTF8);
    return [(NSString *)resultString autorelease];
}

- (NSString *)URLDecodedString
{
    return [self stringByReplacingPercentEscapesUsingEncoding:NSUTF8StringEncoding];
}

@end
```


## ARC Qualifiers

### 题目

原题是这样的:

```
__weak NSString *myString = [[NSString alloc] initWithFormat:@"%@", @"hello"];
NSLog(@"%@", myString);
```

输出结果应该是 `(null)`, 另外还会报错.

### ARC 的四个 qualifier

这些可以在文档中 Transitioning to ARC 的部分 (很久以前的了) 找到, 直接抄过来:

#### __strong

默认的 qualifier. 只要有 strong pointer 指向一个 object, 它就不会被释放.

#### __weak

什么都不做, 只是单纯地指向一个 object, 当没有 strong pointer 指向这个 object 时此 weak reference 被设为 `nil`.

#### __unsafe_unretained

什么都不做, 只是单纯地指向一个 object, 当没有 strong pointer 指向这个 object 时也不会被设为 `nil`.

#### __autoreleasing

用于表示通过 reference (id *) 传递的参数, 并且在 return 时自动释放.

### 为什么经常使用 __weak

使用 `__weak` 最大的好处是当 object 被释放时会被自动设为 `nil`, 这样就避免了可能出现的 `EXC_BAD_ACCESS`.


## Multithreading

随便提一句, 其实很多都是所谓的 queue 而并不是真正地直接管理 threads, 只能说是 threads 的 replacement, 列举出来 (从文档里抄的):

* Operation objects
* Grand Central Dispatch (GCD)
* Idle-time notifications
* Asynchronous functions
* Timers
* Separate processes


## Persistence

几种方式:

* UserDefaults
* Archive (plist)
* SQLite
* Core Data (基于 SQLite)


## alloc init 和 new 的区别

没区别, 只是 Objecitve-C 倾向于显式地写 `alloc` `init`, 在很多 coding guidelines 中也明确要求不能使用 `new`.


## UITableView 重用的原理

当一个 cell 离开可见区域的时候被移除并根据 identifier 放入 `_reusableTableCells`, 当执行 `-dequeueReusableCellWithIdentifier:` 时根据 identifier 从中取出一个 cell 重用, 如果没有可重用的 cell 即返回 `nil`, 则创建一个新的 cell.

关于视图的重用可以参考我之前写的一个简单的 demo: [SlidesDemo](https://github.com/AlexRezit/SlidesDemo)
