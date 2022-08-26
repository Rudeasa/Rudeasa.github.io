---
title: RE
date: 2021-01-06 23:45:55
tags:	
	- re
---

<!--more-->

# RE

## [FlareOn6]Overlong

第一次接触这种题，刚拿到一脸懵逼。

![image-20201218203849450](https://s2.loli.net/2022/03/26/JCduRq7LIMOjhY9.png)

总共只有三个函数，字符串里也没有直接与flag相关的信息。

所以直接看函数吧

![image-20201218204107576](https://s2.loli.net/2022/03/26/smTCHYv6raJN924.png)

主函数很清楚就是将输入的TEXT与`sub_401160`比较，比较28个长度吧。

![image-20201218204346013](https://s2.loli.net/2022/03/26/zUOYx5nMLcw4DbX.png)

![image-20201218204353236](https://s2.loli.net/2022/03/26/zsp19Vkwht27BvA.png)

`sub_401160`里面嵌套一个`sub_401000`函数，而`sub_401000`里面又是很复杂的类似加密函数，一般我看到这些就觉得是自己找的方向不对了（当然也是一般情况下，有时候真的flag就需要去解出很复杂的函数），因为逆向大部分情况下有加解密的函数是不会让你一点都看不懂的，这函数就让我看得有点迷，所以去找别的思路解题。题目名称overlong就是过长，最近也慢慢和pwn打交道，感觉28那个长度可能就是题目关键。毕竟逆向题，七分做三分猜。可是我不咋会修改寄存器的值，上网查阅后使用od（也是我不熟练的软件）

![image-20201218211950835](https://s2.loli.net/2022/03/26/dgYPkQBCEbsxeFi.png)

打开od找到1C（28=0x1C）的位置，修改它的值为FF(反正足够长就行)，然后在运行程序

![image-20201218212157660](https://s2.loli.net/2022/03/26/EhUgdvH9VuJTon6.png)

flag{I_a_M_t_h_e_e_n_C_o_D_i_n_g@flare-on.com}

## [WUSTCTF2020]level3

​			无壳，拉入ida查看主函数

```
int __cdecl main(int argc, const char **argv, const char **envp)
{
  char v3; // ST0F_1
  const char *v4; // rax
  char v6; // [rsp+10h] [rbp-40h]
  unsigned __int64 v7; // [rsp+48h] [rbp-8h]

  v7 = __readfsqword(0x28u);
  printf("Try my base64 program?.....\n>", argv, envp);
  __isoc99_scanf("%20s", &v6);
  v3 = time(0LL);
  srand(v3);
  if ( rand() & 1 )
  {
    v4 = (const char *)base64_encode(&v6);
    puts(v4);
    puts("Is there something wrong?");
  }
  else
  {
    puts("Sorry I think it's not prepared yet....");
    puts("And I get a strange string from my program which is different from the standard base64:");
    puts("d2G0ZjLwHjS7DmOzZAY0X2lzX3CoZV9zdNOydO9vZl9yZXZlcnGlfD==");
    puts("What's wrong??");
  }
  return 0;
}
```

​	很明显有个base64加密，将`d2G0ZjLwHjS7DmOzZAY0X2lzX3CoZV9zdNOydO9vZl9yZXZlcnGlfD==`

直接拿去解密发现不对，然后回来仔细看了看函数。然后发现base解密表被改了

##### ![image-20201204112610072](https://s2.loli.net/2022/03/26/Blszr1Mdpb6xkJ7.png)

直接写脚本，上网搜索发现python Maketrans以及translate()函数用来转换table方便很多

#### Maketrans以及translate()函数

​	[https://blog.csdn.net/xiangshangbashaonian/article/details/81031258?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522160705244719726885840654%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=160705244719726885840654&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_click~default-1-81031258.first_rank_v2_pc_rank_v29&utm_term=Maketrans%E5%87%BD%E6%95%B0]()

​	直接写脚本就可以

```
import base64
import string

str1 = "d2G0ZjLwHjS7DmOzZAY0X2lzX3CoZV9zdNOydO9vZl9yZXZlcnGlfD=="

string1 = "TSRQPONMLKJIHGFEDCBAUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
string2 = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"

print (base64.b64decode(str1.translate(str.maketrans(string1,string2))))

```

就可以直接出flag

wctf2020{Base64_is_the_start_of_reverse}

## [FlareOn3]Challenge1

​	这是和上面差不多一道题型，所以放一块分享。

拿题看主函数

![image-20201218214546257](https://s2.loli.net/2022/03/26/EK2V7MWzqjQkyoe.png)

很明显的str1是我们的输入通过`sub_401260`函数加密后和str2比较得到flag。

而`sub_401260`函数里面这些就可以猜到就是base64加密了

![image-20201218214829159](https://s2.loli.net/2022/03/26/T1GQlsRuIbVyOjp.png)

而且`byte_413000`数组里面应该是换标编码

![image-20201218215017490](https://s2.loli.net/2022/03/26/AHEPk27vyw5jzgK.png)

整理出来得ZYXABCDEFGHIJKLMNOPQRSTUVWzyxabcdefghijklmnopqrstuvw0123456789+/

然后套脚本就可以出来了

![image-20201218215159747](https://s2.loli.net/2022/03/26/MpD14kBOoxyYrvG.png)

flag{sh00ting_phish_in_a_barrel@flare-on.com}

