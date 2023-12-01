# chapter 2 编码以及项目架构
## 🤔1.Unintended variable shadowing 意想不到的变量阴影
variable shadowing：在go中，一个变量在一块区域声明之后还能在内层模块中重新声明
下面的代码，外层内层都定义了client，因为内层用了:=符号声明，所以外层的client是空的，这段代码要不是因为log里面使用了client，一般情况会报错，指示有client变量已声明或者未使用
```
var client *http.Client
if tracing {
  client, err := createClientWithTracing()
  if err != nil {
    return err
  }
  log.Println(client)
} else {
  client, err := createDefaultClient()
  if err != nil {
    return err
  }
  log.Println(client)
}
// Use client
```
那如何保证给原始client变量赋了值呢？两种方式
第一种：在内层模块里面使用临时变量，然后再把临时变量赋给client
![1a51b27ce2e4c4d6b135814a62e9136a.png](:/e01c4cc392ef4a388cb8a5406ceb2a6c)
第二种：在内层模块直接用=赋值


## 🤔2.Unnecessary nested code 没必要的嵌套代码
错误示范：
![8d1a66d547cf86508b73a074031e7d3c.png](:/d7af26730b4f4407ae9debc49609d704)
正确示范：
![ea74202fedb3708f3a4acf292b493f5f.png](:/167c365e1f654504bf3e99cd3d74fafc)
函数嵌套层数越多可读性越差越难理解
if语句如果有返回值，后面就不要用else了
![fc966de6acf96ea0b84153cd45bf2270.png](:/f8892ce0d4c04ca8856d8c886784e7e9)
也可以像下面这样
![4ed9a47cd4591c8ecb91da1d6f48e8d5.png](:/62248a85b3a946bcb44c81b9e752fcad)


## 🤔3.Misusing init functions 滥用init函数
1.main包和redis包，因为main引用了redis中的函数，所以先执行redis包里的init函数，然后执行main中的init函数，最后执行main函数
![97182c4122fd8e78ff11579e30acd711.png](:/ce0cd3481d8f4f1992685bffe5d98177)
2.在同一个包里或者同一个go文件中可以定义多个init函数，一个go文件里init按顺序执行，同一个包不同go，按照go文件命名的字母顺序执行，比如a包里 有a.go,和b.go，先执行a.go再执行b.go中的init
危险在于源文件能够重命名，这会影响执行顺序
![7d7e6f67903c3c5371b9dead4c3c2d22.png](:/59fa652f55514d9991de60bba489bcc2)
**错误示例：**
![51ffe7c93cdeedbd4f733aab954d65b4.png](:/1ca14670370d46bea19714a6052c4e73)
- 首先，init函数里面的容错管理比较局限，因为init没有入参也不返回，唯一能捕获error的方式就是panic，但如果数据库连接出现问题，这样整个程序就会挂掉
- 另一个缺点就是，要写单元测试的时候，init会在测试用例之前执行，如果只是测试一个工具函数，不需要数据库连接，这样init就把单元测试搞复杂了
- 第三个缺点，示例里面数据库连接必须要定义成全局变量，但是全局变量有严重的缺点，包里的任何函数都能修改全局变量，其实更应该封装一个变量，而不是保持它的全局性
应该就把前面的初始化当简单函数处理，错误处理应该交给函数调用者，而且数据库连接池也封装起来了
![84a42ff868a4c45182ac3f62d7979022.png](:/f9aff7cf300345769128acb91db736f4)
**正确示例：**
下面init函数不会报错，因为http.HandleFunc会panic，而且也是有在handle为nil的时候会panic，示例不会有这种情况，同时示例里面也不用创建全局变量，函数也不会影响单元测试，
![e154812f48c1281fba003bd595cdbc39.png](:/0171e5bfaf2e45e08e8cda4bcaafbdfb)
![91790361eebed9b7b1ef4d09f23dc55a.png](:/017f40a44d2d412d918d325cbc69b639)
init一般在定义静态配置的时候有用，大多数情况还是用特定函数来初始化


## 🤔4.Overusing getters and setters / getters和setters过度使用
go里面没有强制的要使用getters和setters获取数据，一般情况下也不需要用，只有在比如后面要添加新功能，要隐藏内部实现更灵活的时候用，用也要按下面的方式用
![5ec3f98ed3788e8a25ecf61cf08756f5.png](:/32a6b9506e6847ad9bf9e64fd21328d5)
![3d31479da460db9ac5004a977733698c.png](:/5fb48e6398e34862bd7c77152534cea1)


## 🤔5.Interface污染
设计接口的时候注意，interface越大抽象能力越弱
用interface的场景：
1.Common behavior 通用功能
比如集合排序，可以抽象为3种方法（Len、Less、Swap），
2.Decoupling 解耦
![538fcf9a1b668b2aab959d492ac2aa95.png](:/f67f9e0f85c84b00b98192d9477019dc)
如果要进行单元测试，需要创建一个新的customer并存储，
如果用上面的方式，因为customerService依赖具体的实现来存储customer，测试的时候不得不需要集成测试，还得启动一个mysql实例
为了更灵活一点，将customerService和具体的实现解耦，通过接口来存储customer，这样测试改方法的时候更灵活，既可以用集成测试，也可以通过模拟使用单元测试
![de34026ac165c5d06991cdc7026ab9f4.png](:/ca011e909879444b9fdba110b39b3241)
3.Restricting behavior  限制行为
示例：
实现一个custom配置包处理动态配置
![842923e4ec5fc6fa77b6cdd915f27065.png](:/bb1c4d14d0bc4588bc7536aae4e142a3)
现在有一个threshold的配置，只能读不能对它更新，如果不想更改配置包怎么实现？
通过创建一个接口来约束对配置的只读功能
![c62e94372b44024689cd29d770010dc8.png](:/f2415295577a4e41afc729c6a744a84d)
Don’t design with interfaces, discover them.
建议接口不要过度设计，需要用的时候再创建，接口过度设计会导致代码很复杂可读性差
![9203e1e9878b4cf5b8c90da086269261.png](:/f3c2c7308bf04be9b0306cc16faa2964)


## 🤔6.生产端的interface
接口应该放在哪？
第一种放在生产侧，指接口的定义和接口功能具体实现放在一个包
第二种放在消费侧，指接口的定义和外部调用接口的客户端放一起
![3850f8a784b1b51bfd498feaf27f638a.png](:/d34a397cd1a54d85b109500e888df742)
在其他语言里使用第一种，但是在go里面一般用第二种方式，前面说过接口不要过度设计，需要用的时候再创建，所以更多的是根据消费侧的需求决定合适的抽象方式
因为客户端可能只需要用接口里的一个方法，这种情况就可以直接在客户端创建一个interface
![87a29acf46cef2abf5e0c6830f963aa7.png](:/af55a13124254c439d42a3746b4354e5)
也有一些场景例外，例如标准库里面，encoding包定义了interface，在子包encoding/json，encoding/binary中实现，这种一般是开发者很清楚这些接口是有用的


## 🤔7.Returning interfaces 返回接口
设计函数的时候，可以返回接口，也可以返回具体实现，如果返回接口会产生循环依赖的问题
![161465edee581900459516182cd0f681.png](:/92e985ceedf04f6e92c7a4be39a6bc73)
store包中InMemoryStore struct实现了在client包中定义的store接口，NewInMemoryStore又返回client的store接口，这样client包就无法调用NewInMemoryStore函数，不然就会产生循环依赖
返回接口会限制灵活性，所以应该
- 返回structs而不是interfaces
- 入参尽可能接收interfaces
在有一种情况下也可以返回interface，就是很明确知道接口对client包有用时
![66935a95e60e830516e6ab91cdb1e648.png](:/ca740d889f4048e69a52dd3d72530ded)


## 🤔8. any 不传达任何信息
any是空接口的别名，可以用来代替interface{}
any缺乏表达能力，后面维护的人还得去看文档或者实现代码才能理解
![08da2cfc6fbed2666ef496a06bed1a8c.png](:/2edefbb0a64c4dc38eeab6a747732427)
编译不会有问题，但是入参和返回值无法表达有用的信息，而且这种可能会有啥类型都调用的情况，比如int
![ba17994af59a4d1581f3fd3f644eff79.png](:/6c11dd8e74ba4764bd219f947c0df27d)
还不如把每个结构体的get和set方法都写一遍，减少不易理解的风险，虽然这样方法会很多，但是一般client包会使用interface，只封装自己需要的方法
![394405ac5eea2f236ee00631cb8cf759.png](:/31b74b92a2b143d0aed6fc53f3c5d6fd)
**可以使用any的场景**
示例1：encoding/json包中的Marshal函数
示例2：database/sql包，如果query字段参数化，也可以用any参数
![887becaad9724be1954cc85efc9a5141.png](:/29e4efeb6cea42b3b3305022b949a534)


## 🤔9. 困惑何时使用泛型
示例：
需要获取`map[string][int]`中的string
![bb7ec2dab52a848694713eaf652a591e.png](:/87ddaef49a4245d090b9a86d5e1d7eb5)
如果后面又新增获取`map[int][string]`的需求，写两个函数或者用switch case的方式都会有代码冗余，而且用switch case方式返回类型必须是any，检查类型是在运行的阶段进行的而不是编译的阶段，所以还得return error，同时调用方还要做类型检查或者额外的类型转换
![324176b575031280366d7bd3dbddbe92.png](:/366a44763e0e4823b03febf4a2f30d13)
使用泛型
![eb1230fa8a30af708e35adeabe9b1f9b.png](:/c62daa1d60a44d7d83aa005cb6312e0a)
注意：map的key必须是可以比较的类型，切片、map和函数都不能直接比较，所有泛型这个的key的类型是comparable而不是any
也可以自定义约束类型
![e483bbae73bb50cbad284909132a6f02.png](:/c01504bde8f84339b37723597c4c2e2a)
调用方式，使用自定义约束类型之后这个函数会强制要求key类型必须是int或者string
![63087319f556c20d39fc674a20e915e4.png](:/f3028d14d6b24b25acf7bdc9a2217172)
**使用泛型的场景：**
1.数据结构：二叉树、堆和链表。。
2.使用any类型切片、map或channel的函数，例如函数要合并两个any类型的channel
![9cc92cdd8350b76ddeb4e4b0fc9ac4bd.png](:/ee958d406a02458d9308ee10fa78064c)
3.提取公用功能而不是变量类型的时候，例如sort包
![fb4d96474fa9a182da8497008235f403.png](:/402bd82f602641988d8525287cbc87d6)
这个接口可以被不同的函数sort.Ints或sort.Float64s使用
![769e8d5f8f82a29add8f2e61d99e6a12.png](:/8490c13d1fec4ae3b3721d09c6374ca5)

## 🤔10. 没有意识到内嵌类型可能出现的问题

## 🤔11. 没有使用 functional options pattern

## 🤔12. 项目不分层

## 🤔13. 创建一些无用的工具包
比如util包，内涵newStringSet和sortStringSet的函数，调用的时候util这个包名毫无意义，应该把包名改成stringset，函数名new和sort就行了，这样也很容易理解

## 🤔14.忽略包名冲突
申明变量的时候变量名和包名冲突，导致包无法再次使用
同时也要注意避免变量名和内置函数的冲突
copy := copyFile(src, dst)

## 🤔15. 没有代码注释

## 🤔16.不使用静态分析器
