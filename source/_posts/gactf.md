---
title: gactf
date: 2020-09-03 16:13:15
tags:
---
最近几天打的比赛复现一下
## Misc
#### SignIn
这道题就是拼图拼一下，扫一下二维码就出来了
![](1.png)
#### Crymisc
这道题我花了蛮久时间的，还是不会，最后是去看了其他师傅的wp，才看懂

他给的是一个.docx文件，把这个文件放到010 Editor里可以看到

![](2.png)

我们可以很清楚的看到里面有PK，这告诉我们这是一个压缩包，就直接把文件的后缀名改成是.zip

![](3.png)

打开这个压缩包，我们能看到里面有两个文件，其中3.jpg是加密的

![](4.png)

![](5.png)

我们先把那个不需要密码的1.txt打开，没有啥有用的信息，这时候我们就要考虑crymisc.zip采取的是伪加密，把它放入010 Editor看到确实是伪加密，那我们只需要将圈起来的地方改为00就破除了伪加密。

![](6.png)

![](8.png)

打开了3.jpg没啥有用的信息，我们就把它还是拖到010 Editor里分析，发现了一段太过规整的字符串，考虑base解密，然后得到了密钥

我们可以发现直接改后缀名解压是行不通的这时候，我们可以选择这段复制到新的十六制文件中，还有不要忘了加文件名PK(50 4B)

![](9.png)

之后我们就可以把crymisc.txt文件解压出来，之后我们打开这个文件就发现了一串emoji

![](7.png)

接下来就要考虑codeemoji解密，这个我是看了Miaotony师傅的题解才知道的，我这里借鉴这个大师傅的解密网址https://codemoji.miaotony.xyz/share.html?data=eyJtZXNzYWdlIjoi8J%2BUrfCfkpnwn5Cw4pyK8J%2BMu%2FCfkKfwn5KZ8J%2BYmPCfjLvwn4228J%2BSkPCfjYzwn4%2BK8J%2BNqfCfmoHwn4%2BK8J%2BRufCfkLbwn5iA8J%2BQtvCfmIDwn5iY8J%2BRufCfkpnwn42C8J%2BSh%2FCfmIDwn5iA8J%2BYqfCfjLvwn42f8J%2BRgvCfjbbwn5KQ8J%2BNjPCfj4rwn42p8J%2BRhvCfj6Dwn5mH8J%2BNgvCfjYLwn5G88J%2BYsfCfmpTwn5C28J%2BRieKcivCfmLHwn4%2Bg8J%2BZh%2FCfjYLwn42C8J%2BRvPCfmLHwn5qK8J%2BYp%2FCfkqjwn5KZ8J%2BSlSIsImtleSI6IvCfmK0ifQ%3D%3D
![](10.png)
