# Projects-Summary
工作中的项目遇到的问题及解决办法总结

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