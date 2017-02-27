---
title: 一种可行的Deeplink简单实现
date: 2017-02-15 20:57:39
tags:
- frontEnd
- js
- deeplink
---

{% asset_img Deep_Link_Measurement.png   &nbsp;%}

前段时间，项目上提了个需求，发给客户的短信要带上一个链接，点击链接时，设备若已安装APP，需要将用户引导至App内的指定页面，若未安装APP，则需要将用户引导至设备相应的商店页面，本以为只是简单Deeplink + URL Schema，但实际上手时发现细节上还是挺麻烦的。

<!-- more -->
### 确定方案
起初，尝试的方案是页面识别设备类型，再根据设备类型调用相应设备APP的URL Schema，Android下直接使用`window.location = URL Schema`的方式，iOS在`window.location`基础上套`setTimeout`，尝试打不开再跳转商店页面的方式，但写完测试发现有几个问题：
- iOS：Safari打开链接时，若设备已安装APP，需要用户点击Open按钮确认才能打开APP，若设备未安装APP，还会弹出Safari的错误alert，体验很不好；
- Android：Chrome上无法使用`window.location`跳转URL Schema，不能唤醒APP；

针对iOS的问题，光前端这边是没辙了，于是和iOS开发一起做了Universal Link的处理，但因为iOS 9以下设备、未安装APP、链接直接在浏览器中打开等几种情况无法触发Universal Link机制，所以链接在作为Universal Link的同时还指向一个静态页面，带有Smart App Banners，如果检测到已安装APP，还是能点击打开，再显示些广告词下载按钮之类宣传，点击下载按钮跳转App Store；
针对Android的问题，问了下Android的开发，得知使用a标签是可以正常跳转的，所以考虑在链接指向的静态页面页面上写了个假的alert弹框，在检测到设备为Android时显示假弹框，询问用户是否要使用APP打开，Open按钮用a标签跳转URL Schema，Cancel按钮隐藏假弹框显示广告词下载按钮，点击下载按钮跳转Play商店页面；

### 设备识别
js做设备识别目前就是`navigator.userAgent`、`navigator.userAgent`，写法多种多样，用得上什么写什么吧
``` JavaScript
    var browser={
        versions:function(){
            var u = navigator.userAgent, app = navigator.appVersion;
            return {//移动终端浏览器版本信息
                trident: u.indexOf('Trident') > -1, //IE内核
                presto: u.indexOf('Presto') > -1, //opera内核
                webKit: u.indexOf('AppleWebKit') > -1, //苹果、谷歌内核
                gecko: u.indexOf('Gecko') > -1 && u.indexOf('KHTML') == -1, //火狐内核
                mobile: !!u.match(/AppleWebKit.*Mobile.*/), //是否为移动终端
                ios: !!u.match(/\(i[^;]+;( U;)? CPU.+Mac OS X/), //ios终端
                android: u.indexOf('Android') > -1 || u.indexOf('Linux') > -1, //android终端或者uc浏览器
                iPhone: u.indexOf('iPhone') > -1, //是否为iPhone或者QQ HD浏览器
                iPad: u.indexOf('iPad') > -1, //是否iPad
                webApp: u.indexOf('Safari') == -1, //是否web应该程序，没有头部与底部
                wechat: u.indexOf('MicroMessenger') > -1 //是否为微信
            };
        }(),
        language:(navigator.browserLanguage || navigator.language).toLowerCase()
    }
```

### 业务流程
因为需求上要求引导用户到APP指定页面，所以链接还需要带上参数id，进入页面后再拿id拼URL Schema
``` JavaScript
    var id = GetQueryString("id")?GetQueryString("id"):'';
    var hostUrl = window.location.host;
    var protocol = window.location.protocol;
    var appUrl = "schema://path/'+id
    openAppFunc();    

    // 网上扒的获取url携带参数方法
    function GetQueryString(name)
    {
        var reg = new RegExp("(^|&)"+ name +"=([^&]*)(&|$)");
        var r = window.location.search.substr(1).match(reg);
        if(r!=null)return  unescape(r[2]); return  '';
    }
```

方法的实现主要就是前面说的判断逻辑，至于HTML就不贴了，在安卓上仿了个iOS的风格的弹窗
``` javascript
function openAppFunc(){
        if(browser.versions.wechat){
            $(".weixin-tip").css("display","block");  //这里做了个针对微信显示提示语的处理
        }else{
            if(browser.versions.android && browser.versions.mobile){
                $('#alertBox').css('display','block');   //针对Android设备显示假弹框
                $(".yesBtn").attr("href",appUrl);  //确定按钮指向Android的URL schema
                $(".noBtn").click(function(){
                    $('#alertBox').css('display','none');   //隐藏假弹框
                    $('#downBox').css('display','block');   //显示广告文字和下载按钮
                })
                //Android也做延时跳转处理，如果设备未安装APP，即使点击确认按钮也会跳转至商店页面
                $(".yesBtn").click(function(){
                    setTimeout(function(){
                        window.location = 'https://Google Play';
                    },500);
                })
                $(".downBtn").attr("href","https://Google Play");   //下载按钮指向Google Play
            }
            else if((browser.versions.iPhone || browser.versions.iPad) && browser.versions.mobile){
                //倒也可以在这里调一次iOS的URL schema，但因为没有做延时跳转，设备未安装APP的情况下，Safari的报错无法自动消失
                //window.location = appUrl; 
                $('#downBox').css('display','block');   //显示广告文字和下载按钮
                $(".downBtn").attr("href","App Store");  //下载按钮指向App Store     
            }
            else {
                $('#downBox').css('display','block');    //显示广告文字和下载按钮
                $(".downBtn").attr("href","https://website");     //下载按钮指向官网
            }
        }
    }
```

### 补充
上面的方案实现之后，效果还行，不过在联调时又遇到了一个不算问题的问题：短信发送的链接本来是在使用Google的短链接服务，不过这样的短链接Universal Link可识别不了，所以只好找笨办法，换用domain.com/s/{id}的形式，也算是“短链接”了，但因为前面做好了不想再改，而且这样的链接也不好拿参数id，于是就从Nginx上下手了，加了条配置——
```
     location ^~ /s/ {                                                                                      
        rewrite  ^/(.+)/(\d+)  /pagename.html?id=$2  permanent;   
    }     
```
让domain.com/s/{id}形式的链接全部转到上面的静态页面，这样，既不耽误Universal Link，Android也能正常使用，可喜可贺，可喜可贺！

