# Minipay 

#### python版本: python3

#### 用django框架实现的简单demo，请参见：https://github.com/dkodoc/minipay_demo

#### 安装方法
方法1：下载压缩包，解压

```bash
tar -xzvf minipay-x.x.x.tar.gz
cd minipay
python setup.py install
```

方法2：
```bash
pip install minipay
```

#### config配置
在项目初始化时，要先加载minipay的配置，如在django中的settings.py中按照以下示例写入配置
```python
from minipay.config import MiniAppsConfig

# 小程序appid
MiniAppsConfig.APP_ID = None 

# 商家mch_id
MiniAppsConfig.MCH_ID = None 

# 小程序secret
MiniAppsConfig.SECRET = None 

# 小程序随机字符串
MiniAppsConfig.NONCE_STR = None

# 小程序KEY，用于解密退款通知
MiniAppsConfig.KEY = None 

# 开发者自己的支付通知接口，比如 https://www.xxxx.com/api/notification/payment
MiniAppsConfig.PAYMENT_NOTIFY_URL = None

#开发者自己的退款通知接口,比如 https://www.xxxx.com/api/nitification/refund
MiniAppsConfig.REFUND_NOTIFY_URL = None

# 微信退款需要用到的商户证书，没有配置的话请求退款会出错
# 详情见：https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=4_3
# 例如
# CERT = '/appclient.pem'
# CERT_KEY = '/appclient_key.pem'
# 申请退款API需要用到的商户证书本地路径，如要使用申请退款API，必须配置以下两个参数
MiniAppsConfig.CERT = None
MiniAppsConfig.CERT_KEY = None

# 模式。参数值有ignore和store， store则必须提供ORM模型类用来保存请求和响应记录, ignore模式不保存记录
# 该功能目前仅能保证和djangoORM兼容
# 该参数可以在使用minipayAPI的时候再传进去，但是同时需要多加一个models参数，用于传入ORM类
# 具体使用在示例中
MiniAppsConfig.DEFAULT_MODE = 'ignore'

# 默认的ORM模型类，可以到用的时候再填
MiniAppsConfig.DEFAULT_MODEL = None

# 默认请求方法 post or get
MiniAppsConfig.DEFAULT_METHOD = 'post'

```
#### API
##### 1 统一下单
```python
import minipay

# 以下三个为必传参数
data = {
    "out_trade_no": "2018112312321321",
    "body": "XX公司-珍珠奶茶",
    "total_fee": "700", # 这里单位是（分） 200=2元
}
unified = minipay.UnifiedOrder(**data)
result = unified.request()
if unified.is_success:
    print("请求成功")
elif unified.is_fail:
    print("请求失败")
```
##### 2 订单查询
```python
import minipay

out_trade_no = 'abv2010102333112'
query = minipay.OrderQuery(out_trade_no=out_trade_no)
response = query.request()
if query.is_success:
    print(response)
    print(query.response_data)
else:
    print(query.error)
    print(response)
```
##### 3 关闭订单
```python
import minipay

close_order = minipay.CloseOrder(out_trade_no="abv2010102333112")
close_order.request()
if close_order.is_success:
    print(close_order.response_data)
else:
    print(close_order.error)

```
##### 4 申请退款
```python
import minipay

refund_fee = 100
total_fee = 200
refund = minipay.Refund(
    out_trade_no="12313123",
    total_fee=total_fee,
    refund_fee=refund_fee
    )
response = refund.request()
if refund.is_success:
    pass
else:
    pass

```
##### 5 退款查询
```python
import minipay

# out_trade_no，out_refund_no，transaction_id三选1
query = minipay.RefundQuery(out_trade_no="asdasd")
query.request()
if query.is_success:
    pass
else:
    pass

```
##### 6 支付成功通知处理
如果要存储通知（支付和退款的）数据，传入的model需要具备out_trade_no字段，否则会一直当作没有处理，而重复存入记录。
```python
# 支付通知处理
# 微信发过来的是XML格式的数据，直接丢进类里面处理即可
# django 示例
from django.http import HttpResponse
from models.models import PayNotice

import minipay

def payment_notification(request):
    notice = minipay.PaymentNotification(
    data=request.body,
    model=PayNotice,
    mode='store'
    )
    response = notice.handle()
    return HttpResponse(response, content_type='application/xml')
```
##### 7 退款成功通知处理
```python
import minipay
from models.models import RefundNotice
from django.http import HttpResponse


def refund_notification(request):
    notice = minipay.RefundNotification(
        data=request.body,
        model=RefundNotice,
        mode='store',
    )
    response = notice.handle()
    return HttpResponse(response, content_type='application/xml')

```
#### 支持在异步框架中使用
```python
import minipay

async def pay(request):
    unified = minipay.UnifiedOrder(out_trade_no=1111, total_fee='12.00', body='test')
    response = await unified.arequest()
    # ...

```
#### BaseMiniPay
BaseMiniPay对象，例如以上的unified、notice、close_order等。有一些属性和方法可以使用。
###### .request()
发起请求动作，返回*.response_data*，如果请求业务失败，则返回*.error*
###### .is_success
请求业务是否成功，成功返回True，失败返回False。
###### .is_fail
请求业务是否失败，是返回True，否返回False。
###### .response_data
微信端返回的响应内容，为dict类型。
###### .error
dict类型，当请求失败时，有错误码code键和错误说明desc键。{"code": "", "desc": ""}。
如果请求成功，则为空字典>
###### .request_data_xml
调用*.request*之后，再调用这个属性。它返回本次请求的内容（xml格式），一般用于在微信签名校验工具
###### .mini_formatted()
返回一个特定格式的字典，用于给小程序wx.requestPaymentAPI调起支付。详情见：https://developers.weixin.qq.com/miniprogram/dev/api/wx.requestPayment.html

