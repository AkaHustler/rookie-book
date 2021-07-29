## 变量

### 变量内部实现

- 变量内部实现

变量有两个组成部分：变量名、变量值，php中将其对应为：**zval，zend_value**，php中的 **变量的内存** 是通过 **引用计数** 进行管理的，而且php7中 **引用计数**是在zend_value而不是在zval上，变量之间的传递、赋值通常是针对zend_value。

1. 变量的基础结构

``` c
//zend_types.h
typedef struct _zval_struct zval;
typedef union _zend_value {
zend_long lval; //int整形
double dval; //浮点型
zend_refcounted *counted;
zend_string *str; //string字符串
zend_array *arr; //array数组
zend_object *obj; //object对象
zend_resource *res; //resource资源类型
zend_reference *ref; //引用类型，通过&$var_name定义的
zend_ast_ref *ast; //下面几个都是内核使用的value
zval *zv;
void *ptr;
zend_class_entry *ce;
zend_function *func;
struct {
uint32_t w1;
uint32_t w2;
} ww;
} zend_value;
```

``` c
struct _zval_struct {
zend_value value; //变量实际的value
  union {
  	struct {
  	ZEND_ENDIAN_LOHI_4( //这个是为了兼容大小字节序，小字节序就是下面的顺序，大字节序则下面4个顺序翻转
  zend_uchar type, //变量类型
  zend_uchar type_flags, //类型掩码，不同的类型会有不同的几种属性，内存管理会用到
  zend_uchar const_flags,
  zend_uchar reserved) //call info，zend执行流程会用到
  } v;
  uint32_t type_info; //上面4个值的组合值，可以直接根据type_info取到4个对应位置的值
  } u1;
  union {
  uint32_t var_flags;
  uint32_t next; //哈希表中解决哈希冲突时用到
  uint32_t cache_slot; /* literal cache slot */
  uint32_t lineno; /* line number (for ast nodes)
  */
  uint32_t num_args; /* arguments number for EX(Thi
  s) */
  uint32_t fe_pos; /* foreach position */
  uint32_t fe_iter_idx; /* foreach iterator index */
  } u2; //一些辅助值
};
```

**zval** 结构比较简单，内嵌一个 union类型的zend_value，保存具体变量类型的值或指针，还有两个union：u1，u2

u1：变量的类型就通过 **u1.v.type** 区分，另一个值 **type_flags** 为类型掩码，在变量的内存管理、gc中会用到

u2：辅助值

从 **zend_value**可以看出，除了 **long、double**类型直接存储值外，其他类型存储的都是指针，指向各自的结构。

2. 类型

标量类型：最简单的类型是 **true、false、long、double、null**，其中true、false、null没有value，直接根据type区分，而long、double的值直接存在value中；zend_long、double，也就是标量类型不需要额外的value指针

字符串：PHP中字符串通过 **zend_string**表示：

``` c
struct _zend_string {
	zend_refcounted_h gc;
	zend_ulong h; /* hash value */
	size_t len;
	char val[1];
};
```

1. gc：变量引用信息，比如当前value的引用数，所有用到 **引用计数** 的变量类型都会有这个结构
2. h：哈希值，数组中计算索引时会用到
3. len：字符串长度，通过该值保证二进制安全
4. val：字符串内容，变长struct，分配时按len长度申请内存

字符串还可分为具体的几类，通过flaf保存：zval.value->gc.u.flags

数组：数组的底层实现就是普通的有序HashTable

对象/资源：对象比较常见。资源指的是tcp连接、文件句柄等，这种类型可以随便定义struct，通过ptr指向

引用：引用实际是指向另外一个PHP变量，对它的修改会直接改动实际指向的zval，通过 **&** 操作符产生一个引用变量，**&** 首先会创建一个 **zend_reference** 结构，内嵌了一个zval，这个zval的value指向 原来zval的value（如果是 **布尔、整形、浮点**则直接复制原来的值），然后将原来zval的类型修改为 **IS_REFERENC**，原zval的value指向新创建的 **zend_reference** 结构。

``` c
struct _zend_reference {
	zend_refcounted_h gc;
	zval val;
};
```

PHP中的引用只可能有一层，不会出现一个引用指向另外一个引用的情况，就是没有C语言中**指针的指针** 的概念

3. 内存管理

如果使用硬拷贝方式，赋值、函数传参硬拷贝一个副本，最终各变量的值是互相独立的，不回出现多个变量共用一个value的情况，但是弊端是效率低，如果是只读操作，硬拷贝的话就会有一份多余数据，这个问题解放方案是：**引用计数+写时复制(copy on write)**，php变量的管理正式基于这两点实现的。

(1)**引用计数**：是指在value中增加一个字段 **refcount** 记录指向当前value的数量，变量赋值、函数传参时，不硬拷贝一份数据，而是 **refcount++**，变量销毁时将 **refcount--**，等到refcount减为0时，表示已经没有变量引用这个value，将它销毁即可。

但从**zend_value**的结构可以看出，并不是所有数据类型都会用到引用计数，**long、double** 直接都是硬拷贝，只有value是 **指针**那几种类型才可能会用到引用计数。

并不是所有的php比变量都会用到引用计数，标量：true/false/double/long/null是硬拷贝自然不需要，除此之外还有两个特殊类型也不会用到：interned string(内部字符串，即上文提到的IS_STR_INTERNED、immutable array)，通过**zval.u1.type_flag** 类型掩码来区分是否支持饮用计数

- **interned string**：内部字符串
- **immutable array**：只有在用opcache的时候才会用到这种类型

(2)写时复制(copy on write)

上面的 **引用计数**，多个变量可能指向同一个value，然后通过 **refcount** 统计引用计数，这时候如果一个变量试图更改value的内容，则会 **重新拷贝** 一份value数据修改，同时断开旧的指向，只有在有必要的时候(写)才会发生硬拷贝。

不是所有类型都可以copy，比如对象，资源，事实上只有 **string、array两种支持**，于引用计数相同，也是通过 **zval.u1.type_flag** 标识value是否可复制。

(3)变量回收

**变量回收**主要有两种：**主动销毁**、**自动销毁**。主动销毁指得就是unset，而自动销毁是php的自动管理机制，在return时减去局部变量的refcount，即使没有显式的return，php也会这么操作，另外就是写时复制时会断掉原来value的指向，这时也会检查断开后旧value的refcount

(4)垃圾回收

php的垃圾回收是根据refcount实现的，当unset、return时会将变量的引用计数减掉，如果refcount减到0则直接 **释放value**，这是简单的gc过程。

``` php
$a = [1];
$a[] = &$a;

unset($a);
```

unset($a)后，由于数组中有子元素指向 $a，所以 **refcount > 0**，无法通过gc机制回收，这种变量就是垃圾，目前垃圾只会出现在 **array、object**这两种类型中，所以目前只针对于这两种情况作处理：

当销毁一个变量时，如果发现减掉 **refcount** 后仍然大于0，且类型是 **IS_ARRAY、IS_ONJECT**则将value放入gc可能垃圾双向链表中，等达到一定数量后，启动 **检查程序**将所有变量检查一遍，如果确定是垃圾则销毁释放

标识变量是否需要回收也是通过 **u1.type_flag** 区分的

### 数组

数组的底层实现为散列表(HashTable，即 **哈希表**)，除了Array类型，内核中也随处用到散列表，比如 **函数、类、常量、已include文件的索引表、全局符号表等** 都用的HashTable存储。

散列表的key-value之间存在一个映射函数，可以根据映射函数直接索引到对应的value值，采用 **直接寻址技术(即 通过key映射到内存地址上的)** ，加快查找速度，查找期望时间为 O(1)

#### 数组结构

php里用数组存放value，而value在数组中的存储位置由映射函数根据key计算确定，映射函数可以采用取模的方式，得到一个整型值，然后于数组总大小取模得到在散列表的位置。

HashTable中另外一个非常重要的值 arData ，这个值指向存储元素数组的第一个Bucket， 插入元素时按顺序 依次插入 数组，比如第一个元素在arData[0]、第二个在 arData[1]...arData[nNumUsed]。**PHP数组的有序性正是通过 arData 保证的**，这是第一 个与普通散列表实现不同的地方。

所以，整体来看HashTable主要依赖arData实现元素的存储、索引。插入一个元素时先将元 素按先后顺序插入Bucket数组，位置是idx，再根据key的哈希值映射到散列表中的某个位置 nIndex，将idx存入这个位置；查找时先在散列表中映射到nIndex，得到value在Bucket数组 的位置idx，再从Bucket数组中取出元素。

#### 映射函数

``` c
nIndex = key->h | ht->nTableMask;
```

位运算要比取模更快

#### 哈希碰撞

哈希碰撞是指不同的key可能计算得到相同的哈希值(数值索引的哈希值直接就是数值本身)， 但是这些值又需要插入同一个散列表。一般解决方法是将Bucket串成链表，查找时遍历链表 比较key。

php也是如此，只是把链表的指针指向转化成了数值指向，即：指向冲突元素的指针并没有直接存在Bucket中，而是保存到了value的 **zval** 中。

当出现冲突时将原value的位置保存到新value的 zval.u2.next 中，然后将新插入的value 的位置更新到散列表，也就是后面冲突的value始终插入header

#### 插入、查找、删除

#### 扩容

PHP散列表的大小为2^n，插入时如果容量不够则首先检查已删除元素所占比例，如果达到 阈值(ht->nNumUsed - ht->nNumOfElements > (ht->nNumOfElements >> 5)，则将 已删除元素移除，重建索引，如果未到阈值则进行扩容操作，扩大为当前大小的2倍，将当前 Bucket数组复制到新的空间，然后重建索引。

#### 重建散列表

重建过程实际就是遍历Bucket数 组中的value，然后重新计算映射值更新到散列表，除了更新散列表之外，这里还有一个重要 的处理：移除已删除的value

### 静态变量

局部变量分配在 **zend_execute_data** 结构上，每次执行 **zend_op_array** 都会生成一个新的 **zend_execute_data** ，局部变量在执行之初分配，然后在执行结束时释放，这是局部变量的生命周期。

局部变量中有一种特殊的类型：静态变量，它们不会在函数执行完后释放，当程序执行离开函数域时静态变量的值被保留下来，下次执行时仍然可以使用之前的值。

### 全局变量

### 常量