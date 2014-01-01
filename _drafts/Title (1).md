---
layout: poslay
title: Little JavaScript Book 19 - syntax joke
label: 酷玩JavaScript
kind:
ptr: JavaScript
mdmark: ran
metakey:
metades:
---

###Joker

    'I\'m a ' + typeof + '';
    //log:  I'm a number
    
又来个毁三观的代码，也不知那个怪咔这么无聊。。第一眼还真把我给唬住了，大家猜猜为啥有真个结果吧。

嗯，我猜你眼就看出猫腻了，没错！第二个加号并不是连接字符串的，而是做**隐式类型转换**的（还记得前面讲的吗？），`+`的一个作用就是将任何类型转化为`Number`类型，而恰恰好这里有个`typeof`运算符，于是乎，typeof得到了`“number”`，之后经过拼接得到`“I'm a number”`。

###R u srsly ？

神奇的事还有不少，下面这个以前没有发现过，
