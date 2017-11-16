---
layout: post
title: "深呼吸后我们再来理清“N维数组，N-1维数组指针，N级指针”之间的纠葛"
date: 2017-10-22
excerpt: "一定要记得深呼吸"
tags: [C++, 指针, 数组]
comments: true
---

# 什么是指针
为了理清“N维数组，N-1维数组指针，N级指针”之间的纠葛，我们需要先了解什么是指针，指针其实就是存放内存地址的一个变量，我们说它指向某个内存地址，除此之外指针我们还得搞清楚这个指针指向的数据类型，因此我们可以认为一个指针有两个域（数值域是内存地址，类型域说明指向的内存数据的类型）。你可能以为指针只要知道内存地址就可以了，其实不然，我们在做指针运算的时候需要跳过指定的指针槽（slot），而一个槽（slot）就等于一个该指针指向的数据类型的字节数。

举个例子

```
int intA[] = {1, 2, 3};
int *p_int = intA;
char charA[] = {'a', 'b', 'c'};
char *p_char = charA;
for(int i = 0; i < 3 ; i++){
    printf("Int:%d, Char:%c; ", *(p_int+i), *(p_char+i));
}
```
你会发现上面的程序是正常运行的，虽然int是四个byte，char是一个byte，但是p_int和p_char这两个指针在进行+1的指针运算的时候都成功指向了下一个数组元素，也就是拿到了下一个数组元素的地址，其实p_int和p_char都跳了一个槽（slot），这个槽的大小刚好等于sizeof(type_of_pointer)，说人话就是这个槽大小就是指针指向的数据的数据类型的sizeof。所以int *指针在加一的时候，实际上内存地址加了四个字节，char *指针在加一的时候，内存地址只加了一个字节。

# 言归正传，我们来研究主题那个三角恋

## 先来看一段代码
> talk is cheap, show me the code

```
int a[2][3] = {1, 2, 3, 4, 5, 6};
const int (*p_a)[3] = a;
int *p_a2 = *a;
//    int *p_a3 = a;  // error
//    int **p_a4 = a; // error

// 遍历方法一
printf("通过a[][]方式遍历二维数组a\n");
for (int i = 0; i < 2; i++) {
    for (int j = 0; j < 3; j++) {
        printf("%d,", a[i][j]);
    }
}

// 遍历方法二
printf("\n通过数组头a遍历二维数组\n");
for (int i = 0; i < 2; i++) {
    for (int j = 0; j < 3; j++) {
        printf("%d,", *(*a + i*3 + j));
    }
}

// 遍历方法三
printf("\n通过数组头a遍历二维数组\n");
for (int i = 0; i < 2; i++) {
    for (int j = 0; j < 3; j++) {
        printf("%d,", *(*(a + i) + j));
    }
}

// 遍历方法四
printf("\n通过p_a遍历二维数组\n");
for (int i = 0; i < 2; i++) {
    for (int j = 0; j < 3; j++) {
        printf("%d,", *(*(p_a + i) + j));
    }
}

// 遍历方法五
printf("\n通过p_a2遍历二维数组\n");
for (int i = 0; i < 2; i++) {
    for (int j = 0; j < 3; j++) {
        printf("%d,", *(p_a2 + i*3 + j));
    }
}

output:
通过a[][]方式遍历二维数组a
1,2,3,4,5,6,
通过数组头a遍历二维数组
1,2,3,4,5,6,
通过数组头a遍历二维数组
1,2,3,4,5,6,
通过p_a遍历二维数组
1,2,3,4,5,6,
通过p_a2遍历二维数组
1,2,3,4,5,6,
```

你会发现"int *p_a3 = a; int **p_a4 = a;"这两句话被编译器给无情拒绝了。为什么呢，课本教我们的难道不是数组头（数组名）可以当作一个常量指针来看待嘛？为什么这里不能赋值给一级指针，可以用来表示二维数组的二维指针也不行呢？数组名a降级为指针话到底是一个什么样的指针呢？它存放的地址应该是多少呢？黑人问号？？？

### 我们先来了解下二维数组的内存存储方式
二维数组在内存中的存储顺序其实是按行存放的，也就是上面代码中的a[2][3] = {1,2,3,4,5,6}数组和a[6] = {1,2,3,4,5,6}在内存中的样子是一模一样的，[]其实是个语法糖，真正的二维数组寻址方式其实是这样的

```
*(*a+i*3+j) 等价于 a[i][j]
也可以写成*(*(a+i)+j)
```
所以数组名a如果降级为指针，应该是一个数组指针，在上面的实例中a是一个指向int [3]的指针，因此为什么const int (*p_a)[3] = a;可以成功编译。所以我们发现a可以看作是指向一个数组（int [3]）的指针，那么 *a 也就是 a[0] 应该是指向a[0][0]的指针，那么烧脑的地方来了，a == *a == a[0] == &a[0][0]！！！

```
printf("\n&a[0][0]:%x, a[0]:%x, *a:%x, a:%x\n", &a[0][0], a[0], *a, a);

output:
&a[0][0]:efbff590, a[0]:efbff590, *a:efbff590, a:efbff590
```

a指针指向的地址居然和 *a 指针指向的地址是一样的！！！什么鬼！！！深呼吸。。。回忆一下我们之前讲过的指针概念，除了数值域（内存地址）还有什么？对，还有指针类型。所以虽然a和 *a 指针指向的内存地址是一样的，但是a是数组指针，它一个槽是3个int也就是12个byte哦，而 *a 也就是a[0] 它是一个int * 指针，一个槽是1个int也就是4个byte。这就解释了为什么"int *p_a3 = a; int ** p_a4 = a;"会报错，根本就不是一个指针类型嘛，当然报错了。

> 不过由于a和*a的值是一样的，a其实可以强转为int *。 int *p_a3 = (int *)a;就ok啦。


# SO..思考一下下面的程序

> 完全可以等量齐观哦
```
int n = 90;
int *p_i_n = &n;
char *p_c_n = (char *)&n;
printf("%x:%d, %x:%c\n", p_i_n, n, p_c_n,p_c_n[0]);

output:
efbff54c:90, efbff54c:Z
```