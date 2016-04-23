Lab2 实验报告
----------------
### 理解和填写代码
#### 较难理解的宏

`page2pa`函数。这个函数是将指向一个页控制块的指针转换为这一页的实地址。因为`pages`数组存放的是所有的页控制块，那么用一个指针减去`pages`得到的就是这个指针所指向的数组元素在`pages`数组中的下标。因为每页的大小是`4K`，所以下标为`0`的元素是地址为`0x00000000`的页的控制块，下标为`1`的元素是地址为`0x00001000`的页的控制块，第`n`个元素控制的页的地址应该是下标左移`12`位。`12`恰好是`PGSHIFT`的值。

`PADDR`、`KADDR`函数和`page2kva`。这两个函数用于虚地址和实地址的互相转换。对于内核存放的`kseg0`段来说，这两个地址只相差`0x80000000`，也就是宏函数中用到的`ULIM`的值。由此也很容易理解`page2kva`函数是将指向某页的控制块的指针转化为这页的虚地址。

`LIST_INSERT_HEAD`函数。第一个参数到底是指针还是结构？为什么传入`page_free_list`会导致编译错误？为了回答这个问题，注意到下面这个表达式
```c
LIST_FIRST((head))->field.le_prev
```
将`LIST_FIRST`宏定义复制到此处，表达式变为
```c
((head)->lh_first)->field.le_prev
```
可见，`head`应是一指向结构的指针，此结构中应有一个名为`lh_first`的指针，此指针应指向另一个结构，此结构中有一个名为`field`的结构，这个结构中有一个成员变量名为`le_prev`。为什么`page_free_list`不符合`head`的要求呢？注意到`page_free_list`的定义
```c
struct Page_list page_free_list;
```
也就是说它是一个结构，这个结构是如何定义的呢？
```c
LIST_HEAD(Page_list, Page);
```
将宏定义替换
```c
struct Page_list {
	struct Page * lh_first;
}
```
至此之前的问题已经解决了，为了彻底的理解，再注意到`Page_LIST_entry_t`和`Page`的定义
```c
typedef LIST_ENTRY(Page) Page_LIST_entry_t;
struct Page {
	Page_LIST_entry_t pp_link;
	u_short pp_ref;
};
```
可见之前由表达式推测的结构内容是正确的。

### 研究思考题
#### Thinking 3.1 使用`do - while(0)`语句的好处

使用`do { } while (0)`代替`{ }`只适用于宏函数没有返回值的情况，好处使得宏函数后总是需要`;`表示结束，与其他函数相同。如果使用`{ }`而多加了`;`在一些情况下会导致严重后果。如
```c
#define FOO() {foo(); bar();}
// ...
if (something)
  FOO();
else do_something;
```
等于
```c
if (something)
  {
    foo();
    bar();
  }
  ;
else do_something; // 编译错误
```
由于多出的分号，`else`语句处出现了编译错误。

#### Thinking 3.2 自映射机制页目录地址的计算
与指导书介绍的例子同理，`0xC0000000`对应的应该是第`0xC0000`个页表项，第一个页表项的地址是`0xC0000000`，第二个页表项的地址是`0xC0000004`，以此类推，第`0xC0000`个页表项的地址应该是`0xC0300000`


#### Thinking 3.3 `NOFOUND`的奥妙
`tlb_invalidate`函数首先判断`curenv`是否存在，而`curenv`在 Lab2 中始终为`NULL`，但是可以在 Lab3 中看到，这个是一个`Env`类型的指针，顾名思义指向的应该是当前进程的进程控制块。`tlb_invalidate`函数会先计算出需要查询的快表项，然后调用`tlb_out`，如果不存在，则直接用页框号调用`tlb_out`。

`tlb_out`首先将`CP0_ENTRYHI`暂时保存到`k1`寄存器中，然后将保存在`ra`寄存器中的参数保存到`CP0_ENTRYHI`中，接下来调用`tlpb`指令，这一指令会查询快表中是否有值为`CP0_ENTRYHI`的项，如果有则保存到`CP0_INDEX`中，如果没有则把`CP0_INDEX`高位置为1，接下来`tlb_out`函数通过`CP0_INDEX`寄存器中的值判断是否找到，如果此值小于`0`说明没有找到，那么跳转到`NOFOUND`，如果找到则使用`tlbwi`指令更新快表。

由于对于进程控制块和进程ID还没有了解，`tlb_invalidate`函数计算快表项的方式我也没有理解。

### 难度评估
这次的实验我总共花费了大约 15 小时。主要的工作是研究代码，也有几小时时间用于撰写实验报告、填写代码和查阅资料。
