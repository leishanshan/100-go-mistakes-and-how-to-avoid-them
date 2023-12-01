# 👀control structures 控制结构篇

## 🤔30.忽视元素在range循环中是值拷贝
**示例：**
![d667230a1a24693f65f59f3456f882e2.png](:/821e89545bc04696a89da8f94a4d8185)
range循环遍历中，会将每个元素进行值拷贝，上面代码中遍历每个account struct会拷贝一个副本赋给a，所以`a.balance+=1000`改变的是临时变量a中的值，原切片中的元素是没有更新的
**解决方式：**
用索引变量 i 访问切片元素，或者用传统循环不用range
![7c18fcaea1c99fd1b55dd14e72663a9d.png](:/f3b027dfdb474a988a2b919f48e6a43c)

## 🤔31.忽视参数在range循环中的计算方式
讨论` for i, v := range exp`中，exp是怎么计算的，exp可以是数组、切片、map或者channel
**示例1：** 下面循环会不会终止？
![fcda5910f48713ee996c17527f43b3f7.png](:/a271735e1d364639b040b2aa7eeeb5d6)
使用range循环，提供的表达式只在循环开始前被计算一次，然后将表达式拷贝到一个临时变量中，再对这个临时变量迭代访问，所以上面的循环迭代3次之后终止，每次迭代中原始切片s也在更新
![d0252ebae45ced99b69edf4f59ae2601.png](:/d9407974ac8446e7a417227856cf35ce)
![d6855a184255833de668e6cff96557d2.png](:/3fff7518fac04a58b9871a2658bb9f62)
用传统循环方式，因为在不断append元素，永远不会终止
![93968c1233098624f1288eab35faf5dc.png](:/bddbe1f2e8484e0ba730ce6ee5f800f9)
**示例2：** 对channel
创建两个goroutine，虽然下面ch2赋给了ch，但是前面先做了ch也就是ch1的拷贝，打印的值是0，1，2
![0516893fafbb2957d907a99886720ce4.png](:/af72c44037c14fd980b5ada5204925fc)
不过ch=ch2也不是没有影响，如果调用close(ch)关闭的是ch2而不是ch1
**示例3：** 对array
![700987ab255d05c20bd7bab17fc90fe3.png](:/3a5edb5a766e4b759efe16c06a976637)
![6ce386dd572958d4a989e4a3e1146561.png](:/54cb79fbadf341c19560af46b689715a)
![ae2d9d5f56c56436245029f283faf4da.png](:/aac9af14767e4930b7e65c647576aea1)
也可以使用切片的指针，这样就不用拷贝整个数组，如果数组非常大，用指针更好一点
![84ea008c9c055c40937657cc51ef375f.png](:/a1b4845d630e4458ac7b481b42d7c60a)

## 🤔32.忽视在range循环中使用指针元素的影响
range循环使用指针元素，如果不够谨慎可能会引用错误的元素
使用指针元素的切片或map的原理：
![4cc50d0b81f492a8891eab9b9f091c4d.png](:/0ca54ba1915c44f58b7d1fbb1fabe071)
1.指针元素被Store结构体和put函数共享
2.有时候已经对指针进行操作了，所以直接在集合存储指针更方便
3.如果store结构体很大而且要频繁修改，用指针可以避免大量的拷贝修改
![768c55ee150c87a4f203a23681893376.png](:/b46d1591174542568dd3b942dd48624e)
**错误示例：**
![c2659f9b882c65ef3187ad6781d5e864.png](:/10d8a7119b68458889669e4b79ee9c8f)
定义了一个Customer结构体，以及Store结构体存储customer信息，storeCustomers对Customer切片用range遍历并将信息存到map中
![97917e1135033251636ce30cca848441.png](:/711820e1c730406c97dc659c24f918c6)
如果调用这个函数，打印map结果会是什么？
![8cc0bd0e8791e1f3cdcccf730c967f6e.png](:/534808b632244623a0920f70c1ef5b1f)
错误原因：用range迭代，创建一个具有固定地址的customer变量
每一次迭代，都是同一个指针指向的不同customer，map中存储的是同一个指针，所以map中的值就是最后一次迭代更新的值
![787d254db0ea55b62b42c59a7e0ed6e6.png](:/ea08873af0214806a4d4c746ae4b7664)
解决：
第一种方式：创建一个局部变量，不存储引用customer的指针，而是存储当前元素的指针，这样map中存储的是引用了不同customer的指针
![a449bd9ef6c30ed12205b6f0194d05e8.png](:/bbc9d108a00e403dbdd1ac67c2a88c4b)
另一种方式：引用customer切片每一个元素的索引指针
![f4c8b6f9bde83be05eeeb73b48199503.png](:/e7feb44e1ba24d2f88ffb9e8724e22b2)

## 🤔33.错误理解Map的迭代
**场景1：排序**
![f5fb1558988bdce114f3992fb692e2ae.png](:/152c8be20de24f3fb0d2ee19094093cb)
map迭代不按照key排序，每次迭代的顺序也可能不一样
**场景2：map迭代时更新元素**
错误示范：
运行多次得到的结果会不一样
![9814e5afe328a24efccf9e191dfffc30.png](:/ef3c60c96d5a437bae44fde85896e35c)
官方有说过，如果遍历map期间创建新的元素，后续的迭代中这个元素可能被修改也可能被跳过
如果需要对map m做更新，正确做法是copy一个副本m2，遍历m，但是对m2作更新

## 🤔34.忽视break的工作原理

## 🤔35.在循环内使用defer
