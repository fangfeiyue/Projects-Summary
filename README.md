# Projects-Summary
工作中的项目遇到的问题及解决办法简略记之

## 技术选型
- 版本控制：git
- 开发工具：vscode
- 软件开发过程：敏捷开发
- 框架选择：React全家桶
- 模块化方案：ES6 + Webpack
- 前后端分离方式：完全分离，纯静态方式
## 社区
### 遇到的问题
1. 问题一：

问题：社区首页选择切换门店地址从生产上的店铺切换到测试店铺，首页数据无变化

原因：因为请求门店地址是异步请求的，还没等数据请求完就先调用了`hashHistory.goBack();`导致门店id号没有及时变化，页面返回首页调用调动`fetchAndHandleExhibition`方法更新首页数据，因为门店id还是之前的id，没能及时更新，所以会从store中读取之前的数据进行渲染。

HomeAddressBarSearchContainer中相关代码
```
this.props.dispatch(fullitem? addressSelectedFull(fullitem):addressSelectedPOI(item, getCurrStore(serverdata.data,'storeId'), getCurrStore(serverdata.data,'storeName'),serverdata.data));
hashHistory.goBack();
```

exhibition.js中相关代码
```
export function fetchAndHandleExhibition(exb_id,shared) {
  return (dispatch,getState) => {
    //情况1 直接命中store
    // console.debug("#$#$#$#$ exb case1, 是否命中store.exhibition");
    if (getState().exhibition) {
      const {lastUpdated,storeId} = getState().exhibition;
      // console.debug('getState().exhibition.storeId:', getState().exhibition.storeId);
      // console.debug('exb_id:',getState().exhibition.exb_id , exb_id);
      if (getCurrentStoreId()
          && getCurrentStoreId() == getState().exhibition.storeId
          && getState().exhibition.exb_id == exb_id
          && !isCacheExpired(lastUpdated))
      {
          // console.debug("#$#$#$#$ exb case1 命中");
          dispatch(fetchExhibitionSuccess(exb_id,
            {
              code:'0',
              data: {...getState().exhibition} ,
            },
            lastUpdated));

        // catpro
        const data = getState().exhibition.booths;

        const catproData = data.find((item) => item.templet_name === 'CATPRO') || null;
        if (catproData) {
          console.log('catpro cached');
          const response = normalize(schema.formatCatproData(catproData), schema.catproSchema);
          const catproId = catproData.id;
          console.log('catproData normalized', response);
          dispatch(fetchCatproSuccess(
            catproId,
            response,
            Date.now()
          ));
        }

        return Promise.resolve();
      }
    }
    //情况2 命中localstorage
    // console.debug("#$#$#$#$ exb case2, 是否命中localstorage");
    const localdata = loadState(exbkey(exb_id));
    if(localdata && !isCacheExpired(localdata.lastUpdated)
        && getCurrentStoreId()
        && getCurrentStoreId() == localdata.storeId
      ){
      // console.debug("#$#$#$#$ exb case2 命中, locaodata=",localdata);
      dispatch(fetchExhibitionSuccess(exb_id,
        {
          code:'0',
          data: {...localdata} ,
        },
        localdata.lastUpdated));

      // catpro
      const data = localdata.booths;
      const catproData = data.find((item) => item.templet_name === 'CATPRO') || null;
      if (catproData) {
      console.log('catpro cached');
        const response = normalize(schema.formatCatproData(catproData), schema.catproSchema);
        const catproId = catproData.id;
        console.log('catproData normalized', response);
        dispatch(fetchCatproSuccess(
          catproId,
          response,
          Date.now()
        ));
      }

      return Promise.resolve();
    }
    // 情况3 全都没有命中
    // console.debug("#$#$#$#$ exb case3,什么都没命中");
    dispatch(fetchExhibitionRequest(exb_id));
    return api.getExhibitionData(exb_id,shared)
      .then((serverdata) => {
        if(serverdata.code == 0){
          dispatch(fetchExhibitionSuccess(exb_id,serverdata, Date.now()));
          saveState(getState().exhibition, exbkey(exb_id));
          return serverdata.data.booths
        }else{
          dispatch(fetchExhibitionFailure(exb_id,serverdata.message));
        }
      })
      // 把其中的catpro数据单独处理
      .then((data) => {
        if(data==null) return;//added by yuanquan , "正在装修，敬请期待"会被显示为未知原因
        const catproData = data.find((item) => item.templet_name === 'CATPRO') || null;
        if (catproData) {
          const response = normalize(schema.formatCatproData(catproData), schema.catproSchema);
          const catproId = catproData.id;
          console.log('catproData normalized', response);
          dispatch(fetchCatproSuccess(
            catproId,
            response,
            Date.now()
          ));
        }
      })
      .catch((error) => {
        dispatch(fetchExhibitionFailure(exb_id,"未知原因"));
      });
  };
}
```

解决办法：等请求完数据后再调用`hashHistory.goBack();`方法
```
if (fullitem){
  this.props.addressSelectedFull(fullitem).then(()=>{
    hashHistory.goBack();
  });
}else{
  this.props.addressSelectedPOI(item, getCurrStore(serverdata.data,'storeId'), getCurrStore(serverdata.data,'storeName'),serverdata.data).then(()=>{
    hashHistory.goBack();
  });
}
```

## 对接小马管家
### 遇到的问题：
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
togglePicker = () => {
  isNetwork();
  //此时下面的代码照样执行
  console.log('弹出时间控件');
};
```
2017年06月05日 星期一   
## 退货退款


1.bug1chorme的移动页调试时双击放大,解决方法:
```
<meta content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=0" name="viewport" />
```

2.setState的新写法  
```
形式：
this.setState((state, props) => { return {  }});

例子：
this.setState(({isPickerShow}) => ({
  isPickerShow:!isPickerShow
}));
```

3.url从localhost换成本地ip，无法访问,解决办法去掉下面函数中的第二个参数，也就是localhost.
```
app.listen(3000, 'localhost',(err) => {
  if (err) {
    console.log(err);
    return;
  }

  console.log('Listening at http://localhost:3000');
});
```

4.今天项目不能热更新了，刷新页不行，只能重新npm start，原因是文件夹的首字母大写了，导致无法更新。解决办法就是将文件夹名小写。

2017年06月18日 星期日

1.e.target.value可以拿到input的value值


2017年06月20日 星期二

1.
```
<Route path="/:id" component={ApplyReturnsContainer} />
这样写，无法直接在浏览器输入其他路由进行切换，需要在它前面凭借参数
可以换成这样的写法：
<Route path="/returns/:id" component={ApplyReturnsContainer} />
```

2017年06月22日 星期四

1.路由跳转replace会重新拉去数据

2.韩培的点击按钮加载动画的组件，需要在request，success，failure中都写上loading

2017年06月23日 星期五

1.datepicker的使用

2017年06月27日 星期二

1.价钱是小数不能用parseInt()转换，需要用parseFloat();

2. H5 App页面 绝对定位 软键盘弹出时顶起底部按钮。解决办法：按钮定位设置为absolute；然后设置z-index：-1

2017年06月28日 星期三

1.小米手机无法安装charles证书，一直报证书找不到，后来安装了UC浏览器才解决啦。

2.逗逗做的页面安卓显示正常，ios上页面会被底部的悬浮按钮挡住。设想的解决办法有二
- 1.被挡住的页面增加个同级div，这个div的高度设置成悬浮按钮的高度
- 2.悬浮按钮增加父级div，悬浮按钮该怎么定位就怎么定位，靠父级div撑开页面

3.做退款列表的时候,使用了map动态生成列表，但一直没有显示列表。因为map方法里，我没有给return。还有动态生成的列表忘记给唯一的key值了，正确写法如下：
```
const ReturnsItemList = ({ lists }) => {
  return (
    <ul>
      <div className="topTitile">退款</div>
      {lists &&
        lists.map((item, index) => <ReturnsItem item={item} key={index} />)}
    </ul>
  );
};
```

2017年06月29日 星期四

1.a标签用js操作跳转
```
location.href='';
```

2.超出的文字显示折行
```
p{
    width:200px;
    overflow:hidden;
    white-space:nowrap;
    text-overflow : ellipsis; 
}
```

3.z-index:-1；无法解决按钮被顶起的bug,可以在window.onresize方法中设置样式
```
let flag = false;
window.onresize = function(){           
    flag = !flag;

    if (flag){
        $('.btn').hide();
    }else{
        $('.btn').show();
    }
};
```

4.原生js获取元素的所有兄弟元素
```
function getSiblings(element) {
    let siblings = [];
    let elements = element.parentNode.children;
            
    elements = Array.from(elements);

    elements.map((item, index) => {
        if (item !==element){
            siblings.push(item);
        } 
    });

    return siblings;
}
let li = document.getElementsByClassName('test')[0];
        
console.log(getSiblings(li));
```

2017年06月30日 星期五

1.react滚动加载，用到的控件：InfiniteScroll
```
滚动加载拼接数据用到的知识点：
let arr1 = [4, 23, 3, 4];
let arr2 = [55, 6, 73, 81];
console.log([...arr1, ...arr2]); //[4, 23, 3, 4, 55, 6, 73, 81]
--------------------------------------------------------------
reducer.js文件的写法：

const initialState = {
  datass: []
}

....
return {
    ...state,
    datass: [...state.datass, ...action.selectReturnsData.data.list]
}
```

2.对象的结构赋值
```
let metaData = {
    title: '房',
    test: [{
        title: '飞跃',
        desc: '一个懂得感恩的人'
    }]
};

let { title, test: [{title: subTitle}]} =metaData;
console.log(title, subTitle); //房 飞跃
```

3.for...in和for...of
- for...in 循环的是数组的索引，对象的属性名
- for...of 循环的是数组的元素,不能遍历对象
```
var student = {
    name:'student',
    score:'100'
};

for (item of student){
    console.log(item); //Uncaught TypeError: student[Symbol.iterator] is not a function
}
```

2017年07月12日

1.react中img标签引入本地图片的方法,注意：webpack要配置url-loader
```
import xxx from './xxx.png'
...

<img src={xxx} />
或者
<img src={require('./xxx.png')} />
```

2.熊欣按照上面的写法一直报错：element.loader.split is not a function，无法正常显示图片，原因是webpack中的loader配置有误
```
{
    test: /\.(png|jpg|jpeg|gif)$/,
    loader: ['url-loader?limit=8192'],
}
改为：
{
    test: /\.(png|jpg)$/, //匹配.png或.jpg结尾
    loader: 'url?limit=8192'
}
```

3.ES6 模块与 CommonJS 模块的差异
- CommonJS 模块输出的是一个值的拷贝，ES6 模块输出的是值的引用。
- CommonJS 模块是运行时加载，ES6 模块是编译时输出接口。

第二个差异是因为 CommonJS 加载的是一个对象（即module.exports属性），该对象只有在脚本运行完才会生成。而 ES6 模块不是对象，它的对外接口只是一种静态定义，在代码静态解析阶段就会生成。

4.this和event.target的区别：

js中事件是会冒泡的，所以this是可以变化的，但event.target不会变化，它永远是直接接受事件的目标DOM元素；
e是事件对象。
event.target表示发生点击事件的元素；
event.target表示发生点击事件的元素；
this表示的是注册点击事件的元素
this 等于 e.currentTarget
this是所有函数原生具有的.进入函数时,this已经直接有了目标对象.
而e.target通过e再寻找target,中转了一下

2017年07月19日 星期三

1.退款金额验证
```
//必须包含两位小数
var reg = /^(([1-9]+)|([0-9]+\.[0-9]{1,2}))$/;
//小数或者整数
var reg = /^([1-9]\d*|0)(\.\d{1,2})?$/;
```

2017年06月06日 星期二  
从微信商城，我的--》点击退货退款---》进入退货退款列表，每进入一次退货退款的商品都会翻倍，但接口返回的数据并没有增加，原因是使用下面的方法将数据进行了缓存，这种写法数组中有可能重复加载
```
returnsData: [...state.returnsData, ...action.selectReturnsData.data.list]
```
解决办法：采用lodash.union处理数组合并
```
import union from 'lodash/union';
returnsData: union(state, action.selectReturnsData.data.list)
```
### 潜客发券
1.js动态添加的DOM元素因为时序的问题导致无法获取到dom元素，致使点击事件失效，jq的解决办法用事件代理，步骤  
- 先找到父元素再给父元素添加on方法
- on方法传递三个参数，第一个参数是事件，第二参数是事件响应的元素，第三个是事件调用的回调函数
```
$('.swiper-wrapper').on('blur','.verification-code', function (e) {
    e.stopPropagation();
    if (validateSMS($('.verification-code').val()) == true) {
        $('.prompt-msg').html('&nbsp;');
    } else {
        $('.prompt-msg').html(validateSMS($('.verification-code').val()));
    }
});
```

2.领取优惠券页面：获取短信验证码成功，不输入验证码，点击领取优惠券，提示：传入参数错误。解决办法：在点击领券的按钮中做判断，判断手机号或者验证码是否为空，如果为空就直接返回，并且给提示信息，如果不为空再调用领取优惠券的接口领券


3.jq给元素添加属性和删除属性的方法，注意是属性不是css样式
```
//添加属性
$('.get-coupon').attr("disabled", "disabled");
//移除属性
$('.get-coupon').removeAttr('disabled');
```

4.通过params动态的改变swiper的值  
```
mySwiper.params.direction = localStorage.getItem('direction');
```

2017年6月9日 星期五

1.rem布局正确的代码。讲解rem布局好的网址：http://www.cnblogs.com/noobfly/p/6207832.html。
```
(function () {
    size();
    window.onresize = function () {
        size();
    }
    function size() {
            var winW = document.documentElement.clientWidth || document.body.clientWidth;
        //7.5是怎么来的呢？当然是根据设计稿的横向分辨率/100得来的，除以100是为了更好的计算，没有任何理由。
        //如果设计稿基于iphone6，横向分辨率为750，body的width为750 / 100 = 7.5rem
        //如果设计稿基于iphone4/5，横向分辨率为640，body的width为640 / 100 = 6.4rem
        //布局时，设计图标注的尺寸除以100得到css中的尺寸
        document.documentElement.style.fontSize =  winW / 7.5 + "px";
    }
})();
```

2.ps打开标尺的快捷键：ctrl+r  

3.ps切图最好都保存为web格式的  

4.可以将设计图缩放到和手机差不多大小的尺寸来对比看做的网页是否符合设计图  

6.从psd中扣取图片的步骤  
- 选中要扣取的图层
- 右键复制图层
- 选择新建图层
- 扣取图层，并存储为web格式

7.文字换行可以在标签里写<br/>

8.不能用js添加position，可以用添加类名的方法加进去

9.shift+tab往前制动，可以用来调整代码和tab相反

2017年06月16日 星期五

1. 处理移动端web开发的时候，页面某些数字会被认为是电话号码
```
<head>     
<meta name="format-detection" content="telephone=no" />     
<meta name="format-detection" content="email=no" /> 
</head>
```

2017年07月03日 星期一

1.潜客发券首页加载就会报错，因为走到了ajax的error方法中，没有适当的处理error方法中的内容。

2.积分商城的bug：积分商城-评价过长，有连续数字或者英文或者特殊字符的未折行，超出屏幕宽度
```
解决方案，css中添加
word-wrap: break-word;
```

2017年07月04日 星期二

1.退款列表跳转申请退款页面，等跳到申请退款页面一直提示未登陆，原因是申请退款页面判断是微信环境才会登陆。解决办法需要让谷歌浏览器模拟微信环境，具体步骤如下：
- 打开浏览器控制台
- 点击右上角的三个竖着排列的小点选more tools->network conditions
- Disable cache打上对勾
- 去掉Select automatically前面的对勾
- Select automatically下面的下拉框选custom
- 输入框中输入： Mozilla/5.0 (iPhone; CPU iPhone OS 8_4 like Mac OS X) AppleWebKit/600.1.4 (KHTML, like Gecko) Mobile/12H143 MicroMessenger/6.2.3 NetType/WIFI Language/zh_CN

2.上面步骤的快捷键：esc

3.按钮要做成两边很尖的感觉
```
border-radius:60%;
```

2017年07月05日 星期三

1.$ajax添加加载动画
```
--------html文件中--------
<div id="loadgif">
　　<img alt="加载中..." src="https://imgcdn.guoanshequ.com/pad/kkx3x3kg3roit3hoebbhg75qns95wm7t.gif"/>
</div>
--------js文件中----------

$.ajax({
type: 'POST',
url: url,
data: b64string,
headers: headers,
dataType: 'json',
async: false,
beforeSend: function(){
  $('#loadgif').show();
},
success: function (response) {

  console.debug('接口返回的信息--->',response);
},
error: function (error) {

  if (flag == 'info'){
    $('#failureReason').html('请求失败，请稍后再试');
    $('.getFailureNo').css({'display': 'block','opacity': '1','zIndex': '999'});
  }else{
    $('.prompt-msg').html('请求失败，请稍后再试');
  }
},
complete: function(){
  setTimeout(function() {
    $('#loadgif').hide();
  }, 1000);
}
});
```

2017年07月06日 星期四

1.$.ajax()怎么获取服务器端的时间
```
success: function (response, status, xhr) {
  console.log('status--->', status); //success
  let timer = xhr.getResponseHeader('Date');
  timer = new Date(timer);
  console.log('timer--->', timer.getTime());
}
```

2017年07月10日 星期一

1.element.addEventListener(event, function, useCapture)
```
useCapture:
可选。布尔值，指定事件是否在捕获或冒泡阶段执行。
可能值:
true - 事件句柄在捕获阶段执行
false- false- 默认。事件句柄在冒泡阶段执行
```

2.命名空间
```
(function(window, document){

    function gotoApp(){
        
    }
    
    window.addEventListener(
        'DOMContentLoaded',
        function(){
            document
            .getElementById('appBtn')
            .addEventListener('click', gotoApp, false);
        },
        false
    );
})(window, document);


-------------------我是华丽的分割线-------------------
$(function(){
    
}) == $(document).ready(function(){});
```

3.html打开app
```
/**
 * ios universal link
 * android wechat 应用宝链接
 * android 浏览器 schema
 *
 * 使用
 * html中，给a或者button, 绑定id "gotoGASQApp"
 *
 */

(function(window, document) {
  // android
  var android_schema = 'launcher://com.guoan.app';
  var android_yingyongbao_url = 'http://a.app.qq.com/o/simple.jsp?pkgname=com.guoan.app&android_schema=' + encodeURIComponent(android_schema);

  // ios
  var ios_universalLink = 'https://download.guoanshequ.com/download.html';
  var ios_yingyongbao_url = 'https://itunes.apple.com/us/app/guo-an-she-qu/id1037699537';

  // download url
  download_url = 'https://download.guoanshequ.com/download.html';

  // timeout
  var timeout = 600;

  function checkBrowser() {
      // 根据ua判断平台
      var ua = navigator.userAgent;
      var isApp = ua.indexOf('guoanshequ') > -1;
      var isAndroid = ua.indexOf('Android') > -1 || ua.indexOf('Adr') > -1; // android终端
      var isiOS = !!ua.match(/\(i[^;]+;( U;)? CPU.+Mac OS X/); // ios终端
      var isWechat = ua.indexOf('MicroMessenger') > -1; // wechat
      var isChrome = ua.match(/Chrome\/([\d.]+)/) || ua.match(/CriOS\/([\d.]+)/); // adnroid chrome
      return {
        isAndroid: isAndroid,
        isiOS: isiOS,
        isWechat: isWechat,
        isChrome: isChrome,
        isApp:isApp,
      };
    }

  function gotoApp() {
    console.log('gotoApp');
    var ua = checkBrowser();

    var isAndroid = ua.isAndroid,
      isiOS = ua.isiOS,
      isWechat = ua.isWechat;
      isApp = ua.isApp;
    if(isApp){
      var uri='';
      if(isAndroid){
        uri = 'launcher://com.guoan.app/pg$.pop';//android
      }else{
        uri = 'guoanshequ://pg$.pop';//ios
      }
      window.location.href=uri;
      return;
    }
    // 不同情况区分
    // android微信，跳应用宝链接
    if (isAndroid && isWechat) {
      window.location.href = android_yingyongbao_url;
      return;
    }

    // ios微信 跳 universal link, 需要在了download.guoanshequ.com/download.html页面识别 配应用宝链接
    if (isiOS && isWechat) {
      console.log('isiOSWechat');
      window.location.href = ios_universalLink;
      return;
    }

    // ios浏览器 跳universal link
    if (isiOS && !isWechat) {
      console.log('isiOS');
      window.location.href = ios_universalLink;
      return;
    }

    // android 非微信浏览器，用隐藏iframe跳schema
    if (isAndroid && !isWechat) {
      openAppBySchema();
      return;
    }

    window.location.href = 'https://www.guoanshequ.com';
  }

  // iframe openapp
  function openAppBySchema() {

    var startTime = Date.now();
    var ua = checkBrowser();

    if (ua.isChrome) {
      // 一些 android chrome 浏览器  无法从iframe唤起，采用a标签唤起
      var aLink = document.createElement('a');
      aLink.href = android_schema;
      aLink.style.display = 'none';
      document.body.appendChild(aLink);
      aLink.click();
    } else {
      // 从iframe唤起schema
      var iframe = document.createElement('iframe');
      iframe.src = android_schema;
      iframe.style.display = 'none';
      document.body.appendChild(iframe);
    }

    // 计时器延迟来判断是否启动成功
    var timerId = setTimeout(function() {
      var endTime = Date.now();

      if (!startTime || endTime - startTime < timeout + 200) {
        window.location = download_url;
      } else {
      }
    }, timeout);

    // cleartimeout
    window.onblur = function() {
      clearTimeout(timerId);
    };
  }

  // event binding
  window.addEventListener(
    'DOMContentLoaded',
    function() {
      document
        .getElementById('gotoGASQApp')
        .addEventListener('click', gotoApp, false);
    },
    false
  );

  window.addEventListener(
    'DOMContentLoaded',
    function() {
      document
        .getElementById('appBtn')
        .addEventListener('click', gotoApp, false);
    },
    false
  );
})(window, document);

```
### hybrid相关 h5与native交互方法说明
```

已实现的基本交互(未来可用后面的替代)

●设置顶部导航标题
setNativeTitle(string)

●设置地址
setFullAddress(json)


h5调用native方法

http:///gasq/uploadimg?

统一方法名 callNativeHandler
根据传的jsonData数据区分不同的原生方法
●iOS
    window.webkit.messageHandlers.callNativeHandler.postMessage(JSON);

●Android
    window.JSBridge.callNativeHandler(JSON)

JSON 数据格式：
{"handlerName":"uploadImage","data":null,"callbackId":"callback_1_1497518893399”}

handlerName: h5调用原生的方法名，此例中是调用原生”uploadImage”的功能
data: h5传给原生的参数，没有传null
callbackId: 回调函数id, 没有回调则不传

具体方法名称约定：
方法名
作用描述
回调数据
uploadImage
上传单张图片： 调用原生图库/拍照，
String 图片url，
scanCode
扫码：启动原生摄像头扫码(快递单)
String 具体编码


native调用h5方法

统一方法 JSBridge.handleMessageFromNative(JSON)

作为h5调用原生后的回调使用

在js调用原生方法callNativeHandler后，原生处理完数据，回调js时使用 JSBridge.handleMessageFromNative(JSON)

JSBridge.handleMessageFromNative({"responseId": "callback_1_1497518893399", "responseData": “url...”})

JSON数据格式：
{"responseId": "callback_1_1497518893399", "responseData": "123abcdef”}


responseId: 回调函数id，需要跟h5调用原生方法callNativeHandler时传的callbackId一致
responseData: 回调数据，原生处理后需要返给h5的数据，嵌套数据转JSON

作为直接调用h5方法使用

原生直接调用h5暴露的方法

JSBridge.handleMessageFromNative({"handlerName": "setData", "data": "1234abcd”})

JSON数据格式：
{"handlerName": "setData", "data": "1234abcd”}

handlerName: h5在暴露给原生来调用的方法名
data: 原生传输给h5的数据 JSON格式

具体方法名称约定：

方法名
作用描述
setData
客户端给h5页面传输数据,数据格式为JSON，一般是进入h5页面后传输header相关数据(token, 各种id等等)


————————

reference

> Hybrid APP基础篇(四)->JSBridge实现原理与示例 | Dailc的个人主页 ★★★ 基本按照这个实现的
```

### 客户端与h5数据交互接口约定
hybrid相关 h5与native交互方法说明

退货退款

🔗 退款URL
申请退款: “#"必须要有
测试2(dev)：http://wx.guoanshequ.ren/dev-wx_js_bundle/returns/#/{orderId}_{orderItemId}
测试： https://wx.guoanshequ.wang/returns/#/{orderId}_{orderItemId}
预生产： https://wx.guoanshequ.top/returns/#/{orderId}_{orderItemId}
生产：https://wx.guoanshequ.com/returns/#/{orderId}_{orderItemId}
退款详情: “#"必须要有
测试2(dev)：http://wx.guoanshequ.ren/dev-wx_js_bundle/returns/#/returnsDetails/{orderAftersaleId}
测试：https://wx.guoanshequ.wang/returns/#/returnsDetails/{orderAftersaleId}
与生产：https://wx.guoanshequ.top/returns/#/returnsDetails/{orderAftersaleId}
生产：https://wx.guoanshequ.com/returns/#/returnsDetails/{orderAftersaleId}

🍿 客户端传给h5的页面初始化数据

进入申请退款页面
token
appTypePlatform

        JSBridge.handleMessageFromNative(JSON)
        JSON：
            {
              "handlerName": "setData",
              "data": {
                    "token": "customer_app_66c1ad70796073db1f3691c7c1592ec9",
                    "appTypePlatform": "ios"
              }
            }

进入退款详情页面
token
appTypePlatform

        JSBridge.handleMessageFromNative(JSON)
        JSON：
            {
              "handlerName": "setData",
              "data": {
                    "token": "customer_app_66c1ad70796073db1f3691c7c1592ec9",
                    "appTypePlatform": "ios"
              }
            }


🤝 接口约定

h5调用客户端

方法名 | 作用描述 | 回调数据
---|---|---
uploadImage | 上传单张图片： 调用原生图库/拍照，|必填，String 图片url，取消传空字符串
scanCode | 扫码：启动原生摄像头扫码(快递单) | String 具体编码
applyReturnsSuccess | 发起退货退款申请成功后，跳转到客户端对应订单的退款页面 | 无
setPageTitle | 设置页面标题 | 无


客户端调h5
方法名 | 作用描述
---|---|---
setData|客户端给h5页面传输数据,数据格式为JSON，一般是进入h5页面后传输header相关数据(token, 各种id等等)

5.微信上传图片：调用上传，调用php接口上传阿里，返回阿里url，浏览器显示图片

6.微信分享和原生定的规则
```
从h5界面跳转到app或者微信商城（h5需要判断是ios设备还是安卓设备）
iOS                guoanshequ://gd$.505911111111111(商品和服务回传的时候都传gd)
Android             launcher://com.guoan.app/gd$.505911111111111(商品ID)
```

7.refs
- 以前的写法
```
this.refs.myTextInput.focus();
<input type="text" ref="myTextInput" />
```
- 现在的写法
```
//调用
this.textInput.focus();
//声明
<input ref={(input) => { this.textInput = input; }} />
```

2017年07月11日 星期二

1.CSS3 calc() 会计算的属性，calc是英文单词calculate(计算)的缩写，是css3的一个新增的功能，你可以使用calc()给元素的border、margin、pading、font-size和width等属性设置动态值。

width: calc(100% - 8rem);

2.设置0.5px宽度的边框
```
li::after {
    content: '';
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
    height: 1px;
    background: #ddd;
    -webkit-transform: scaleY(0.5);
    -moz-transform: scaleY(0.5);
    -ms-transform: scaleY(0.5);
    -o-transform: scaleY(0.5);
    transform: scaleY(0.5);
    -webkit-transform-origin: 0 0;
    -moz-transform-origin: 0 0;
    -ms-transform-origin: 0 0;
    -o-transform-origin: 0 0;
    transform-origin: 0 0;
}
```
## 说明
如果对您有帮助，您可以点右上角 "Star" 支持一下 谢谢！ ^_^

或者您可以 "follow" 一下，我会不断开源更多的有趣的项目
## 个人简介
作者：房飞跃

博客地址：[前端网](http://www.qdfuns.com/house/31986/note)  [博客园](https://www.cnblogs.com/fangfeiyue)  [GitHub](https://github.com/fangfeiyue)

职业：web前端开发工程师

爱好：探索新事物，学习新知识

座右铭：一个终身学习者

## 联系方式
坐标：北京

QQ：294925572

微信：

![XinShiJieDeHuHuan](http://note.youdao.com/yws/public/resource/c2361265179a03449f6d52397fd50033/xmlnote/100D55934BB446839482D3EA0CDC3E8D/17820)

## 赞赏
觉得有帮助可以微信扫码支持下哦，赞赏金额不限，一分也是您对作者的鼎力支持

![微信打赏](http://note.youdao.com/yws/public/resource/c2361265179a03449f6d52397fd50033/xmlnote/D77744C8EC944CF6AA232272CBC5CF6D/17828)
