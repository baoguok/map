# 路书
骑行路线规划应用

一个基于 [高德地图API](https://lbs.amap.com/api/javascript-api-v2/documentation) 制作的地图 web 应用，可以分享路线、查看 gpx 路径、地图信息点分享、各市区县布局等。

路线分享  
[http://kylebing.cn/tools/map/#/route/route-line](http://kylebing.cn/tools/map/#/route/route-line)  
<img width="2032" alt="Screenshot 2024-01-23 at 17 45 28" src="https://github.com/KyleBing/map/assets/12215982/19597ade-9f78-4fdc-a0db-b0fd78fca6b4">

gpx 路径 展示  
[http://kylebing.cn/tools/map/#/gpx/gpx-viewer](http://kylebing.cn/tools/map/#/gpx/gpx-viewer)  
<img width="2032" alt="GPX" src="https://github.com/KyleBing/map/assets/12215982/293ac9e9-dc52-4224-b855-97c707cf07ea">

gpx 3D 路径展示  
[http://kylebing.cn/tools/map/#/gpx/gpx-viewer-3D](http://kylebing.cn/tools/map/#/gpx/gpx-viewer-3D)  
![2024-05-06 13-52-05 2024-05-06 13_55_08](https://github.com/KyleBing/map/assets/12215982/1bd3033e-84cf-4d56-9b3f-22e6c46c94b8)

数据点展示  
[http://kylebing.cn/tools/map/#/pointer/pointer-viewer](http://kylebing.cn/tools/map/#/pointer/pointer-viewer)  
<img width="2032" alt="Screenshot 2024-01-23 at 17 39 57" src="https://github.com/KyleBing/map/assets/12215982/9ef18183-a004-4e51-a547-49bcb9fdfbd9">

范围标记  
[http://kylebing.cn/tools/map/#/tool/circle](http://kylebing.cn/tools/map/#/tool/circle)  
<img width="2032" alt="Screenshot 2024-01-23 at 17 40 29" src="https://github.com/KyleBing/map/assets/12215982/6deab80c-ebcd-49db-a3d9-c4d178869682">

市内区县展示  
[http://kylebing.cn/tools/map/#/tool/district-info](http://kylebing.cn/tools/map/#/tool/district-info)  
<img width="2032" alt="Screenshot 2024-01-23 at 17 40 42" src="https://github.com/KyleBing/map/assets/12215982/5a1603cb-1ecf-4782-a22d-5e684cfc3653">

---

# 开发

## 一、使用

修改 `/src/mapConfig.js` 中的高德地图开发 key，获取地址： [https://console.amap.com/dev/key/app](https://console.amap.com/dev/key/app)

有两个，别选错了


<img width="646" alt="开发api key 版本" src="https://github.com/KyleBing/map/assets/12215982/f3c7a06c-08b6-42b1-b4fc-a56569b925b4">


```js
const key_web_js = ''   // web js key
const key_service = ''  // web服务 key

export {
    key_web_js,
    key_service
}
```

Web JS 在使用的时候还需要在服务器中设置对应的安全密钥，具体参见官方文档：
> [JS API 安全密钥使用 https://lbs.amap.com/api/javascript-api-v2/guide/abc/jscode](https://lbs.amap.com/api/javascript-api-v2/guide/abc/jscode)

## 二、后台程序

后台：[https://github.com/KyleBing/portal](https://github.com/KyleBing/portal)


## 三、用到的技术
- `vue3` + `pinia` + `router` 
- `ts`
- 高德 API 2.0

## 四、历程
- 2021-06-28 init `vue2`
- 2024-07-26 `vite` + `ts` + `vue3`


## 五、todo
- [x] 点过多时，列表滚动 `2025-04-18`