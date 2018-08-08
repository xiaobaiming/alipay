AliPay SDK for Golang


## 鸣谢

感谢下列人员对本项目的支持：

[@wusphinx](https://github.com/wusphinx)

[@clearluo](https://github.com/clearluo)

[@zwh8800](https://github.com/zwh8800) 

## 帮助

在集成的过程中有遇到问题，欢迎加 QQ 群 564704807 讨论。

## 已实现接口

#### 手机网站支付API

* **手机网站支付接口**
	
	alipay.trade.wap.pay
	
* **电脑网站支付**

	alipay.trade.page.pay

* **统一收单线下交易查询**
	
	alipay.trade.query
	
* **统一收单交易支付接口**
	
	alipay.trade.pay
	
* **统一收单交易创建接口**

	alipay.trade.create
	
* **统一收单线下交易预创建**

	alipay.trade.precreate
	
* **统一收单交易撤销接口**

	alipay.trade.cancel
	
* **统一收单交易关闭接口**

	alipay.trade.close

* **统一收单交易退款接口**

	alipay.trade.refund
	
* **App支付接口**

	alipay.trade.app.pay

* **统一收单交易退款查询**

	alipay.trade.fastpay.refund.query

* **单笔转账到支付宝账户接口**

	alipay.fund.trans.toaccount.transfer
	
* **查询转账订单接口**

	alipay.fund.trans.order.query 
	
#### 通知
	
* **通知内容转换及签名验证**
	
	将支付宝的通知内容转换为 Golang 的结构体，并且验证其合法性。
	
## 集成流程

从[支付宝开放平台](https://open.alipay.com/)申请创建相关的应用，使用自己的支付宝账号登录即可。

#### 沙箱环境

支付宝开放平台为每一个应用提供了沙箱环境，供开发人员开发测试使用。

沙箱环境是独立的，每一个应用都会有一个商家账号和买家账号。

#### 应用信息配置

参考[官网文档](https://docs.open.alipay.com/200/105894) 进行应用的配置。

本 SDK 中的签名方法默认为 **RSA2**，采用支付宝提供的 [RSA签名验签工具](https://docs.open.alipay.com/291/105971) 生成秘钥时，建议秘钥的格式采用 **PKCS1**，秘钥长度采用 **2048**。所以在支付宝管理后台请注意配置 **RSA2(SHA256)密钥**。

生成秘钥对之后，将公钥提供给支付宝（通过支付宝后台上传）对我们请求的数据进行签名验证，我们的代码中将使用私钥对请求数据签名。

请参考 [如何生成 RSA 密钥](https://docs.open.alipay.com/291/105971)。

#### 创建 Wap 支付

``` Golang
var aliPublicKey = "" // 可选，支付宝提供给我们用于签名验证的公钥，通过支付宝管理后台获取
var privateKey = "xxx" // 必须，上一步中使用 RSA签名验签工具 生成的私钥
var client = alipay.New(appId, partnerId, aliPublicKey, privateKey, false)

var p = AliPayTradeWapPay{}
p.NotifyURL = "http://xxx"
p.ReturnURL = "http://xxx"
p.Subject = "标题"
p.OutTradeNo = "传递一个唯一单号"
p.TotalAmount = "10.00"
p.ProductCode = "FAST_INSTANT_TRADE_PAY"

var url, err = client.TradeWapPay(p)
if err != nil {
	fmt.Println(err)
}

var payURL = url.String()
fmt.Println(payURL)
// 这个 payURL 即是用于支付的 URL，可将输出的内容复制，到浏览器中访问该 URL 即可打开支付页面。
```

#### 同步返回验签

支持自动对支付宝返回的数据进行签名验证，详细信息请参考[自行实现验签](https://doc.open.alipay.com/docs/doc.htm?docType=1&articleId=106120).

如果需要开启自动验签，只需要在初始化 AliPay 对象的时候提供 **aliPublickKey** 参数，该参数的值为支付宝管理后台获取到的支付宝公钥，如下：

``` Golang
var client = alipay.New(appId, partnerId, aliPublickKey, privateKey, false)
```

#### Return URL

发起支付的时候，当我们有提供 Return URL 参数，那么支付成功之后，支付宝将会重定向到该 URL，并附带上相关的参数。

```Golang
var p = AliPayTradeWapPay{}
p.ReturnURL = "http://xxx/return"
```

这时候我们需要对支付宝提供的参数进行签名验证，当然，前提是我们在 alipay.New(...) 初始化方法中有正确提供 **支付宝公钥**：


```Golang
var client = alipay.New(appId, partnerId, aliPublickKey, privateKey, false)

http.HandleFunc("/return", func(rep http.ResponseWriter, req *http.Request) {
	req.ParseForm()
	ok, err := client.VerifySign(req.Form)
	fmt.Println(ok, err)
}
```

#### 验证支付结果

有支付或者其它动作发生后，支付宝服务器会调用我们提供的 Notify URL，并向其传递会相关的信息。参考[手机网站支付结果异步通知](https://doc.open.alipay.com/docs/doc.htm?spm=a219a.7629140.0.0.XM5C4a&treeId=203&articleId=105286&docType=1)。

我们需要在提供的 Notify URL 服务中获取相关的参数并进行验证:

```Golang

var client = alipay.New(appId, partnerId, aliPublickKey, privateKey, false)
 
http.HandleFunc("/alipay", func(rep http.ResponseWriter, req *http.Request) {
	var noti, _ = client.GetTradeNotification(req)
	if noti != nil {
		fmt.Println("支付成功")
	} else {
		fmt.Println("支付失败")
	}
})
```

此验证方法适用于支付宝所有情况下发送的 Notify，不管是手机 App 支付还是 Wap 支付。

#### 关于合作者身份ID (partnerId)

支付宝提供了验证通知有效性的方法，详细情况可以参考 [https://docs.open.alipay.com/58/103597/](https://docs.open.alipay.com/58/103597/)。

本 SDK 中的验证示例如下：

```Golang

var client = alipay.New(appId, partnerId, aliPublickKey, privateKey, false)
 
http.HandleFunc("/alipay", func(rep http.ResponseWriter, req *http.Request) {
	var noti, _ = client.GetTradeNotification(req)
	if noti != nil {
		if ok := client.NotifyVerify(noti.NotifyId); ok {
			fmt.Println("支付成功")
		} else {
			fmt.Println("不是支付宝发送的通知")
		}
	} else {
		fmt.Println("支付失败")
	}
})
```

验证通知有效性的时候，需要提供合作者身份ID，如果不需要调用 NotifyVerify() 方法进行通知有效性的验证，那么就可以不用传递 partnerId 参数。

如何查看合作者身份ID，请参考 [https://docs.open.alipay.com/58/103544/](https://docs.open.alipay.com/58/103544/) 

#### 支持 RSA 签名及验证
默认采用的是 RSA2 签名，如果需要使用 RSA 签名，只需要在初始化 AliPay 的时候，将其 SignType 设置为 alipay.K\_SIGN\_TYPE\_RSA 即可:

```Golang
var client = alipay.New(...)
client.SignType = alipay.K_SIGN_TYPE_RSA
```

当然，相关的 Key 也要注意替换。

## License
This project is licensed under the MIT License.