# 👀Strings 字符篇

## 🤔36.不理解rune的概念
rune是int32的别名
type rune = int32
rune关键字把字符串转成对应的unicode值
utf-8 把unicode的两个字节，拆成3个字节，并填充上utf8标志位
![eb32525b3efec0cd36aaec9fa8b22e51.png](:/80941086ab404c428f1e3656f68cb52a)

## 🤔37.错误的字符串迭代
![b998db4ccd6f5fc5ea54c56d377fd0b6.png](:/7fc835da430f49128939c7873dcb21d4)
首先，len返回字符串中的字节数而不是符文数量
其次，遍历s时，打印s[i]不打印每个符文，而是打印一个符文的每个起始索引
![d291acb830d62210d4b4acf25016b533.png](:/e89acb6d42f3453a811eb3e1711e253b)
正确遍历方式：
```
//方式1.使用range循环遍历
s := "hêllo"
for i, r := range s {
	fmt.Printf("position %d: %c\n", i, r)
}
//方式2.将字符串转成rune
s := "hêllo"
runes := []rune(s)
for i, r := range runes {
	fmt.Printf("position %d: %c\n", i, r)
}

s := "hêllo"
r := []rune(s)[4]
fmt.Printf("%c\n", r) // o
```


## 🤔38.错误使用trim函数
TrimRight和TrimSuffix函数经常会被混淆
TrimRight/TrimLeft移除前后在给定set中的元素
TrimSuffix/TrimPrefix 移除指定的前缀/后缀

```
//结果是123，因为TrimRight会遍历123oxo中的每个字符，如果是xo中的一部分就删除
fmt.Println(strings.TrimRight("123oxo", "xo"))
//结果是123o，TrimSuffix
fmt.Println(strings.TrimSuffix("123oxo", "xo"))

fmt.Println(strings.TrimLeft("oxo123", "ox")) // 123
fmt.Println(strings.TrimPrefix("oxo123", "ox")) /// o123
```


## 🤔39.字符串连接concatenation的使用优化不足
错误示范：
```
func concat(values []string) string {
s := ""    
for _, value := range values {
	s += value                   //每次迭代更新s时都会重新分配一块内存，性能不高
}
return s
}
```
正确示范：使用strings包中的Builder构造
```
func concat(values []string) string {
sb := strings.Builder{}
for _, value := range values {
	_, _ = sb.WriteString(value)  
}
return sb.String()
}
```
![0b08bd2531a05bb7f22a9c3b8108a47e.png](:/0cb3f977e09f4178ad44ca1331c6e8ce)
使用 strings.Builder也可以append byte、rune
![50557b7a0144d4593599ea0b32995716.png](:/cddc922f79d54cbfa4e80c3c21e2e5c7)
此外，strings.Builder还提供了一个Grow(n int)方法
```
func concat(values []string) string {
	total := 0
	for i := 0; i < len(values); i++ {
		total += len(values[i])          //对每个字符串遍历计算字节总数
	}
	sb := strings.Builder{}
	sb.Grow(total)
	for _, value := range values {
		_, _ = sb.WriteString(value) 
	}
	return sb.String()
}
```
![e08c3a8692e2de59f8eed485ee69237b.png](:/719a5e7bd4ac4858b364b7e922550add)
综合：如果字符串切片大小不超过5，使用操作符+=更好，如果切片长，最好还是用strings.Builder，如果能预先知道字节数，使用Grow方法来预先分配内部字节片

## 🤔40.无用的字符串转换

## 🤔41.子串和内存泄漏
