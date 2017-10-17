# Projects-Summary
工作中的项目遇到的问题及解决办法简略记之

# 对接小马管家
## 技术选型
- 版本控制：git
- 开发工具：vscode
- 软件开发过程：敏捷开发
- 框架选择：React全家桶
- 模块化方案：ES6 + Webpack
- 前后端分离方式：完全分离，纯静态方式
## 遇到的问题：
1.是小马管家商品但没有code，时间选择控件偶尔报indexOf is undefined。复现步骤：先选择是小马管家的商品带有code到下单页，然后在选择不带code的到下单页

重新过了遍代码，发现每次点击立即购买跳转到下单页时，上一次点击的立即购买跳转到下单页中的state依然保留，没有进行清空，此时如果上一次是小马管家的商品，时间控件填充的是小马管家的时间，而这次不是小马管家的商品，再选取小马管家的时间就会报错

解决办法：每次点击立即购买跳转到下单页时都清空原来的state
```
//container中调用
this.props.resetCheckOrder();

//action写法
export const RESET_CHECK_ORDER = 'RESET_CHECK_ORDER';
export const resetCheckOrder = () => ({
  type: RESET_CHECK_ORDER,
});

//Reducer中
case RESET_CHECK_ORDER:
      return {};
```

2.立即购买有自定义编码的商品，填写订单页服务时间默认应是：请选择预约时间，目前是直接显示年月日

因为要实时获取小马管家的服务时间，哪怕在订单页服务时间也有可能变化，所以需要当用户点击选择时间后才去请求后台接口，不能跟之前一样用户到下单页直接获取到服务时间，此时就需要对之前的时间控件加以改造,此处还要和普通时间进行区分。
```
思路：
    用户点击选择时间
        调用后台接口请求服务时间
            请求成功
                更新时间控件的内容并弹出控件供用户选择
            请求失败
                提示用户请求失败信息，不弹出时间控件
```
3.断网的情况下点击请选择预约时间，应该弹出提示信息，如：网络开小差，请稍后再试！
```
if (!navigator.onLine){
  this.props.dispatch(createToastAndAutoDismiss("网络开小差，请稍后重试！"));
  return;
}
```
在写这个的过程中犯了一个特别低级的错误，我把它封装了一个函数，在另一函数中调用,当没网的时候调用函数中下面的代码不执行，可结果是原来的代码照样执行，错误犯在我糊涂的以为在isNetwork中return掉，在调用函数中也就return掉了。。。。
```
const isNetwork = () => {
  if (!navigator.onLine){
    this.props.dispatch(createToastAndAutoDismiss("网络开小差，请稍后重试！"));
    return;
  }
};
```

3.断网的情况下点击强选择
3.断网的情况下点击强
