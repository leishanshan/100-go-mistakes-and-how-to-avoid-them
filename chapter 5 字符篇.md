# 👀Strings 字符篇

## 🤔36.不理解rune的概念
rune是int32的别名
type rune = int32
rune关键字把字符串转成对应的unicode值
utf-8 把unicode的两个字节，拆成3个字节，并填充上utf8标志位

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/4ec2d414-db12-40c2-9bc3-f2ca4e251f0d)


## 🤔37.错误的字符串迭代

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/92fb58b5-9c1e-487a-91bf-5659f767e734)

首先，len返回字符串中的字节数而不是符文数量
其次，遍历s时，打印s[i]不打印每个符文，而是打印一个符文的每个起始索引

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/24f7ad86-e589-44e2-b2a1-c3e7987b985e)

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

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/f27854d3-9ecc-4598-aa99-a91543ec54d6)

使用 strings.Builder也可以append byte、rune

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/ed464001-b024-4d7e-a824-650716d27c22)

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
![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/89ad95ba-1fbb-4bf6-a920-0da03e806671)

综合：如果字符串切片大小不超过5，使用操作符+=更好，如果切片长，最好还是用strings.Builder，如果能预先知道字节数，使用Grow方法来预先分配内部字节片

## 🤔40.无用的字符串转换

## 🤔41.子串和内存泄漏
