## 🤔48.使用panic
panic可以直接结束函数执行流并返回上一个堆栈直到协程被返回或者被recover捕获（recover只能用在defer中，因为defer函数会继续执行即便函数panic了）
panic用来标记真正的异常场景，比如程序错误
![c2dc85868aee1034dfafe9bf62939422.png](:/b499dae4b8af4e708ebd952f96264097)
另一个使用场景：
程序需要依赖项但无法初始化时，例如公开一个服务需要验证电子邮件地址，用了一个正则表达式，这种情况下正则表达式是一个强制性的依赖项，如果不能编译它后面就无法验证任何电子邮件输入，所以必须使用panic以防出现错误

大部分场景下应该使用error返回错误

## 🤔49.忽视什么时候需要包装error
从Go1.13起，%w可以直接允许我们去包装error
场景1：添加额外的上下文到error
eg：“Permission denied” => “user X access resourceY cause permission denied”
场景2：将error标记为某种具体的error
eg：http处理程序，检查调用函数时收到的所有error时是否都是Forbidden类型，这样就可以返回403，这种场景就可以将error包装在Forbidden中
%w和%v的区别：
使用%w，源error依然有效，源error的类型和值都能unwrap，但是使用%v后源error无效
![a120e5487b9739c2df4caf94cda33807.png](:/f11f3e6999fe4261b76b2d2c82c6796c)
![bc988acc45a8ee3561feadabaeed05cc.png](:/5d8173503c8940dfa209179809f2c768)
但是使用%w也有局限性，例如使用调用者Foo检查源error是否是bar error，后面如果将实现做了修改并使用另一个返回其他类型error的函数，Foo检查函数就会有问题，为了确保客户端不依赖我们所考虑的实现细节，返回的错误应该被转换而不是被包装，用%v更合适
![0c6fcc42086680d05659d0150da689bb.png](:/554be8c7869b4624a9814bd945375fa2)

## 🤔50.错误的检查error类型
场景：处理http请求，需要根据id从数据库中取交易金额并返回，会出现两种失败情况，1是id无效，2是db查询失败
对第1种情况，想返回400，对第2种想返回503
```
type transientError struct {
	err error
}
func (t transientError) Error() string {
	return fmt.Sprintf("transient error: %v", t.err)
}
func getTransactionAmount(transactionID string) (float32, error) {
	if len(transactionID) != 5 {  
		return 0, fmt.Errorf("id is invalid: %s",transactionID)    //ID无效返回简单error
	}
	amount, err := getTransactionAmountFromDB(transactionID)
	if err != nil {
		return 0, transientError{err: err}    //db查询失败返回transientError
	}
	return amount, nil
}


//http handler
func handler(w http.ResponseWriter, r *http.Request) {
	transactionID := r.URL.Query().Get("transaction")
	amount, err := getTransactionAmount(transactionID)
	if err != nil {
		switch err := err.(type) {            
		case transientError:
			http.Error(w, err.Error(), http.StatusServiceUnavailable)
		default:
			http.Error(w, err.Error(), http.StatusBadRequest)
		}
		return
	}
	// Write response
}

```
缺点：如果重构getTransactionAmount，transientError由getTransactionAmountFromDB返回而不是由getTransactionAmount返回，如果运行这段代码，总是会返回400
```
func getTransactionAmount(transactionID string) (float32, error) {
	// Check transaction ID validity
	amount, err := getTransactionAmountFromDB(transactionID)
	if err != nil {
		return 0, fmt.Errorf("failed to get transaction %s: %w",transactionID, err)
	}
	return amount, nil
}

func getTransactionAmountFromDB(transactionID string) (float32, error) {
	// ...
	if err != nil {
		return 0，transientError{err: err}
	}
	// ...
}
```
![62875afefd10e5bcaf9fa60d88585e9a.png](:/3b8ed3cc4e3c4e8391d5dd47abda92df)
重构后：
![b914c6f0f3abed8d7d939d05b75c6130.png](:/7229990b05b24d499b4dca3afdf82b75)
解决方式： errors.As
```
func handler(w http.ResponseWriter, r *http.Request) {
	// Get transaction ID
	amount, err := getTransactionAmount(transactionID)
	if err != nil {
		if errors.As(err, &transientError{}) {
			http.Error(w, err.Error(),http.StatusServiceUnavailable)
		} else {
			http.Error(w, err.Error(),http.StatusBadRequest)
		}
		return
	}
	// Write response
}
```
总结：如果要使用error wrapping，就必须要用 errors.As来检查error是否是指定类型


## 🤔51.错误的检查error值
哨点错误：用全部变量定义的error
场景：一个返回行切片的查询方法，如果查询不到，如何处理？一种是返回nil切片，另一种是返回客户端会检查的指定的错误，对于后面这种情况，通常需求是允许请求查询不到rows的，可以归类为预期错误（网络问题、连接错误等这些就是意外的错误），就可以用哨点错误指定。比如标准库里sql.ErrNoRows--查询不到任何rows的时候返回的错误
意外错误应该设计为错误类型：type BarError struct { … }
预期错误应该被设计为错误值：哨点错误 
```
var ErrFoo = errors.New("foo")
```
使用==操作符比较
```
err := query()
if err != nil {
	if err == sql.ErrNoRows {
		// ...
	} else {
		// ...
	}
}
```
当然，如果要用fmt.Errorf和%w包装哨兵error，就不能用==比较了，使用 errors.Is
```
err := query()
if err != nil {
	if errors.Is(err, sql.ErrNoRows) {
	// ...
	} else {
	// ...
	}
}
```

## 🤔52.处理error两次
```
func GetRoute(srcLat, srcLng, dstLat, dstLng float32) (Route, error) {
	err := validateCoordinates(srcLat, srcLng)
	if err != nil {
		log.Println("failed to validate source coordinates")
		return Route{}, err
	}
	err = validateCoordinates(dstLat, dstLng)
	if err != nil {
		log.Println("failed to validate target coordinates")
		return Route{}, err
	}
	return getRoute(srcLat, srcLng, dstLat, dstLng)
	}
func validateCoordinates(lat, lng float32) error {
	if lat > 90.0 || lat < -90.0 {
		log.Printf("invalid latitude: %f", lat)
		return fmt.Errorf("invalid latitude: %f", lat)
	}
	if lng > 180.0 || lng < -180.0 {
		log.Printf("invalid longitude: %f", lng)
		return fmt.Errorf("invalid longitude: %f", lng)
	}
	return nil
}
```

## 🤔53.不处理error

## 🤔54.不处理defer errors
