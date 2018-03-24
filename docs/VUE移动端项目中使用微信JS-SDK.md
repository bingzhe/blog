前段时间在vue移动端项目中有使用到微信的js-sdk,做下记录并分享，可能还会有一些疏忽的地方，如有发现希望可以指出。

## 安装引入

和其他的库都一样，并没有什么特别。

    //安装
    npm install weixin-js-sdk --save
    //使用
    import wx from "weixin-js-sdk";
    

## 配置

通过config接口注入权限验证配置，**所有需要使用JS-SDK的页面必须先注入配置信息**，vue项目是单页面应用，所有只需要在最开始配置一次，其他需要的地方就可以调用相应的接口。

jsapi_ticket是企业号号用于调用微信JS接口的临时票据，生成签名之前必须先有这个jsapi_ticket，通过access_token来获取存在后台，所以把签名用到的字段传给后台，后台签名后再把appId和signature返给前台配置
```
    //签名要的字段
            let noncestr = Util.GetRandString(16);
            let timestamp = moment().format('X');
            let url = window.location.href;
            url = url.split("#")[0];
    //ajax获取后台签名,这里就省略取的ajax过程，可以拿到appId，signature，
    
    //配置，这里配置了分享朋友圈和分享的接口
            wx.config({
             debug: false, // 开启调试模式,调用的所有api的返回值会在客户端alert出来，若要查看传入的参数，可以在pc端打开，参数信息会通过log打出，仅在pc端时才会打印。
             appId: appid, // 必填，公众号的唯一标识
             timestamp: timestamp, // 必填，生成签名的时间戳
             nonceStr: noncestr, // 必填，生成签名的随机串
             signature: sign, // 必填，签名，
             jsApiList: ['onMenuShareTimeline', 'onMenuShareAppMessage'] // 必填，需要使用的JS接口列表，
           });
```    

## 分享

微信的分享并不能直接分享出去，只是修改了自定义了分享内容，还是需要用微信的分享按钮
```
        //修改分享给朋友,分享朋友圈也类似
        shareMessage() {
            let url = "";
    
            wx.onMenuShareAppMessage({
                title: this.shop_name, // 分享标题
                desc: "分享描述", // 分享描述
                link: url, // 分享链接，该链接域名或路径必须与当前页面对应的公众号JS安全域名一致
                imgUrl: "", // 分享图标
                // type: "", // 分享类型,music、video或link，不填默认为link
                // dataUrl: "", // 如果type是music或video，则要提供数据链接，默认为空
                success: function () { },
                cancel: function () { }
            });
        };
```    

## vue中使用

所有的想过方法封装到一个对象里，在Vue入口的生命周期中调用，类似这样
```
        import WxConfig from "@/config/wxConfig";
    
        new Vue({
            el: '#app',
            router,
            store,
            template: '<App/>',
            components: { App },
            created: () => {
                //自定义微信分享
                let wxConfig = new WxConfig();
                wxConfig.config();
                wx.ready(() => {
                    wxConfig.shareMessage();
                    wxConfig.shareTimeline();
                });
            }
        })
```    

## 使用图像接口

使用wx的接口，没有找到如何从微信图像接口中选到图片，并且取到图片的base数据。从而实现自己上传，只好从其他思路解决。 首先用选图接口选到图片得到，然后localIds，在用上传图片接口和localIds上传微信的服务器，得到serverId，后台在用serverId和微信的临时素材接口把图片保存到自己的服务器上，并传给前台图片名。

获取临时素材接口 https://qyapi.weixin.qq.com/cgi-bin/media/get?access_token=ACCESS_TOKEN&media_id=serverId
```
        //头像选择
        chooseAvater() {
            return new Promise((resolve) => {
                wx.chooseImage({
                    count: 1, // 默认9
                    sizeType: ['compressed'], // 可以指定是原图还是压缩图，默认二者都有 'original',
                    sourceType: ['album', 'camera'], // 可以指定来源是相册还是相机，默认二者都有
                    success: function (res) {
                        var localIds = res.localIds[0]; // 返回选定照片的本地ID列表，localId可以作为img标签的src属性显示图片
                        resolve(localIds);
                    }
                });
            });
        }
    
        //微信上传图片
        getImageUpload(localIds) {
            return new Promise((resolve) => {
                wx.uploadImage({
                    localId: localIds, // 图片的localID
                    isShowProgressTips: 1, // 默认为1，显示进度提示
                    success: function (res) {
                        var serverId = res.serverId; // 返回图片的服务器端ID
                        resolve(serverId);
                    },
                });
            });
        }
    
        //后台下载图片再返回前台
        wxUploadImg(serverId){
            return new Promise((resolve) => {
                let data = {
                    wx_upload:1,
                    media_id:serverId
                }
    
                wxImgSave(data).then(resp=>{
                    if(resp.ret === 0){
                        let filename = resp.data.filename;
                        resolve(filename)
                    }
                });
            });
        }
    
       //需要的页面调用
    
          let wxConfig = new WxConfig();
    
          wxConfig
            .chooseAvater()
            .then(wxConfig.getImageUpload)
            .then(wxConfig.wxUploadImg)
            .then(resp => {
              this.headerImage = resp;
              其他操作
            });
```

## 链接

[文档](http://qydev.weixin.qq.com/wiki/index.php?title=%E5%BE%AE%E4%BF%A1JS-SDK%E6%8E%A5%E5%8F%A3)