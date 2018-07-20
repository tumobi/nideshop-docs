# 微信登录和支付

微信小程序登录和支付功能需要配置 weixin 字段信息，必须保证和你的账号一致，否则登录失败和支付失败

配置文件为 src/common/config/config.js

```
// default config
module.exports = {
  default_module: 'api',
  weixin: {
    appid: '', // 小程序 appid
    secret: '', // 小程序密钥
    mch_id: '', // 商户帐号ID
    partner_key: '', // 微信支付密钥
    notify_url: '' // 微信异步通知，例：https://www.nideshop.com/api/pay/notify
  }
};
```

如果只需要登录功能，只需要填写 appid，secret，如果获取请查看官方文档：https://developers.weixin.qq.com/miniprogram/dev/index.html