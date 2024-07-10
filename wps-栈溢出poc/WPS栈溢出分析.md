## 概述



WPS Office是由位于珠海的中国软件开发商金山软件开发的，适用于Microsoft Windows，macOS，Linux，iOS和Android的办公套件。WPS Office由三个主要组件组成：WPS Writer，WPS Presentation和WPS Spreadsheet。个人基本版本可以免费使用。WPS Office软件中存在一个远程执行代码漏洞，该漏洞是由Office软件在分析特制Office文件时不正确地处理内存中的对象引起的。成功利用此漏洞的攻击者可以在当前用户的上下文中运行任意代码。故障可能导致拒绝服务。易受攻击的产品WPS Office，影响版本11.2.0.9453。



## 漏洞分析



在WPS Office中用于图像格式解析的Qt模块中发现堆损坏。嵌入WPS office的特制图像文件可能会触发此漏洞。当打开特制的文档文件时，将触发访问冲突。EDX指向数组的指针，而EAX是指向数组的索引。



![图片](https://mmbiz.qpic.cn/mmbiz_png/kFJsMia2yBica986ajwXHm8HO0m2WuonIRMMfiaH5Y5efj2g1oQXJnNZuiaecufPuf6uKB94saPfx7hWSYMjOvrt9g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



崩溃是如何触发的？让我们看一下PNG标头格式。



![图片](https://mmbiz.qpic.cn/mmbiz_png/kFJsMia2yBica986ajwXHm8HO0m2WuonIR3pkDGWP659RUicYm5cWXACmO1wkG8Fdj19NSfSSJaQicuSgyibtFPXLfw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



从偏移量0x29E31开始-0x29E34是PNG文件格式的签名标头。PNG头文件的结构：



![图片](https://mmbiz.qpic.cn/mmbiz_png/kFJsMia2yBica986ajwXHm8HO0m2WuonIRy8ae966vw60k7SGKwgPmr4QDAkdScZzZt5RJlnqTHiaB0bHxPSRUKfw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



在这种情况下，当WPS Office Suite中使用的QtCore库解析PLTE结构并触发堆损坏时，该漏洞位于Word文档中的嵌入式PNG文件中。在偏移量0x29E82到0x29E85处，调色板的解析失败，从而触发了堆中的内存损坏。崩溃触发之前的堆栈跟踪：



![图片](https://mmbiz.qpic.cn/mmbiz_png/kFJsMia2yBica986ajwXHm8HO0m2WuonIRRzm6XQvdQrUmRkgvZGExAjz8A8Y4ADeGwrzB4aYM80yUicSS5wmTfKQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



在QtCore4解析嵌入式图像之前，我们可以看到来自KSO模块的最后一个调用，试图处理图像kso！GdiDrawHoriLineIAlt。使用IDA Pro分解应用程序来分析发生异常的功能。最后的崩溃路径如下（WinDBG结果）：



![图片](https://mmbiz.qpic.cn/mmbiz_png/kFJsMia2yBica986ajwXHm8HO0m2WuonIRN0Bmg5raCGQTnX8ISbsFW2gOXl3Jmrl7hzvgjavWR6PwUFQHfG5seQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



在IDA Pro中打开时，我们可以按以下方式反汇编该函数：



![图片](https://mmbiz.qpic.cn/mmbiz_png/kFJsMia2yBica986ajwXHm8HO0m2WuonIRQMiaKDgcKSib7yyTH5FLpzz43JuDyzesC8e8xU0hGbiaLonohBO2iaUHcQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



使用故障转储中的信息，我们知道应用程序在0x67353321（移动eax，[edx + eax * 4 + 10h]）处触发了访问冲突。我们可以看到EAX寄存器由0xc0值控制。因此，从这里我们可以根据导致异常的指令对寄存器的状态进行一些假设。需要注意的重要一点是，在发生异常之前，我们可以看到ECX（0xc0）中包含的值被写入到以下指令所定义的任意位置：



![图片](https://mmbiz.qpic.cn/mmbiz_png/kFJsMia2yBica986ajwXHm8HO0m2WuonIREVaf8FD59yVQm1G5fGFUqdSqyFGVkC2zH6Vcubu7wxbnZdcMSyUic9Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



此外，我们注意到，在我们的故障指令之后，EBP的偏移量存储在ECX寄存器中。我们在前面提到的指令（偏移量为0x6ba1331c）上设置了一个断点，以观察内存。断点触发后，我们可以看到第一个值c45adfbc指向另一个指针，该指针应该是指向数组的指针。





![图片](https://mmbiz.qpic.cn/mmbiz_png/kFJsMia2yBica986ajwXHm8HO0m2WuonIRTLqhDOJ7icabodyplVsNKjoU7mvWaL6niavb6NJWEKThZtM5sicibAATIQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



观察c45adfbc的内存引用，发现另一个指针。第一个值ab69cf80始终表示为指向它所引用的任何地方的指针。指针ab69cf80基本上是我们指针的索引数组。





![图片](https://mmbiz.qpic.cn/mmbiz_png/kFJsMia2yBica986ajwXHm8HO0m2WuonIRpFw0jSzoSUQyjk7OrA0icw3Z8wT6m0ddhnQO78ywbkNcJNRXhjZMOZQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



因为我们知道崩溃的路径，所以我们可以使用下面的命令简单地设置一个断点。该命令将获得指针值“ edx + eax * 4 + 10”，并检查其是否满足0xc0。





![图片](https://mmbiz.qpic.cn/mmbiz_png/kFJsMia2yBica986ajwXHm8HO0m2WuonIR86ibnDcHnvVSN6xCnmQG60qKVa0oTrlrB2jtbu90rXXARl5HaQK7eqg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



如果观察堆栈，可以看到以下执行：





![图片](https://mmbiz.qpic.cn/mmbiz_png/kFJsMia2yBica986ajwXHm8HO0m2WuonIReMnuZ4uUPVo7Ez8NqTsXXwJ8OHMBQlVBxNVFK5mOAsDFlqw2vR1ibAQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



如果我们反汇编6ba3cb98，则可以看到以下反汇编代码。真正的根本原因在于此代码。



![图片](https://mmbiz.qpic.cn/mmbiz_png/kFJsMia2yBica986ajwXHm8HO0m2WuonIRyKXTnkOFEJ2Wz7ichVib3NP5uZiazHDib6nRxzaMACSrjXK2us64YRz1bA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



我们在6ba3cb89地址上设置断点并观察ESI + 0xB4，我们可以看到一个指针指向另一个位置：





![图片](https://mmbiz.qpic.cn/mmbiz_png/kFJsMia2yBica986ajwXHm8HO0m2WuonIRclPyZIwmEqquPliasT7NQ9sQicR86Bx9NHRfWSjesRYBE8ERa69U9O4g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



从这里我们可以知道代码实际上没有从指针中释放任何东西。一旦移至EDX，EDX将保留指向索引数组的指针：



![图片](https://mmbiz.qpic.cn/mmbiz_png/kFJsMia2yBica986ajwXHm8HO0m2WuonIRiadH4kd1ZfS6t4SJSPOCqLQqX9ib1ZIsXyWCUlz5oqJZgpt6Kj2N5icRA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



崩溃后的堆栈跟踪：



![图片](https://mmbiz.qpic.cn/mmbiz_png/kFJsMia2yBica986ajwXHm8HO0m2WuonIReZXAibcwnMqq4KKibbBUyUFSH58d2m4hhnibTuBEw0eeRd79qjxWpuI9Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



堆分析：



![图片](https://mmbiz.qpic.cn/mmbiz_png/kFJsMia2yBica986ajwXHm8HO0m2WuonIRmqibpXC3m4OGIydgR2iaDZXx1tKxZQac0iaC11VANFQyaRrQ1icm9ac20A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



段中的最后一个堆条目通常是一个空闲块。堆块的状态指示为空闲块。堆块声明前一个块的大小为00108，而当前块的大小为00a30。前一块报告其自身大小为0x20字节，不匹配。位置为05f61000的堆块的使用似乎是该堆块的使用导致了后续块的元数据损坏的可能性。堆块：





![图片](https://mmbiz.qpic.cn/mmbiz_png/kFJsMia2yBica986ajwXHm8HO0m2WuonIREYp7PalQkzMvmNUfOTkVs2LZmwyKbLibdOPFHsEmGRxauYPylxCpx6A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)