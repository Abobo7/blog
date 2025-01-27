---
title: il2cppdumper实战——XX方舟
date: 2023/7/24

---



---

虽然zygisk-il2cppdumper已经能基本满足需要，但动态dump出来的文件只能看个方法名，没有我们需要的dll、ida分析脚本等，所以还是需要使用il2cppdumper进行dump来获取。

准备工作：GG修改器、粥的客户端、IDA（用来看偏移）、il2cppdumper

### 1.GG修改器导出so

因为粥的so做了加密，直接打开so并不能找到所需要的字符串，所以需要从内存中导出

![image-20230724111535500](https://s2.loli.net/2023/07/24/F9tahKklpw35GqA.png)

![image-20230724111602691](https://s2.loli.net/2023/07/24/rDGqRBT4eL9ivnp.png)

### 2.IDA分析导出的so

把导出的文件丢进ida，加载完按shift+f12转到字符串页面，搜索global_metatdata.dat

按X查找引用，发现只有一个函数，直接跳转过去

![image-20230724112113752](https://s2.loli.net/2023/07/24/oISDlTbEWVux1zv.png)

对比源码可知dword_937B00C即为s_GlobalMetadataHeader，具体可参考此帖

https://www.52pojie.cn/thread-1748681-1-1.html

![image-20230724112348639](https://s2.loli.net/2023/07/24/zbmQWk5Zt49j8EM.png)

### 3.GG修改器跳转

使用偏移计算器，将libil2cpp.so的基址加上偏移找到s_GlobalMetadataHeader

![image-20230724112719660](https://s2.loli.net/2023/07/24/gJUclVuDEoB3rRK.png)

长按进行指针跳转

![image-20230724112751974](https://s2.loli.net/2023/07/24/oGcEF1qCkSWQ6be.png)

这些就是global-metadata.dat，把它dump出来，修改文件头为AF 1B B1 FA即可

### 4.il2cppdumper进行dump

具体方法不再赘述，这里简单讲一下怎么找il2CppCodeRegistration和il2CppMetadataRegistration

对比反汇编的源码和unity源码，可以找到它们所对应的值。在我的版本中，分别是dword_937AFFC

和dword_937B000。具体请自行分析。

找到这两个之后，在GG修改器用基址加上偏移定位到内存，再进行一次指针跳转即可得到

![image-20230724113408191](https://s2.loli.net/2023/07/24/VLwDz4ijx5QJOEg.png)

在il2cppdumper的Auto模式失效的情况下手动输入即可

![image](https://s2.loli.net/2023/07/24/l468yO13smEtrLG.png)