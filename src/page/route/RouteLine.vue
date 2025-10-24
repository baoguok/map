<template>
    <div class="map-container">
        <FloatingButton
            :show="!isRouteListShowed && store.isInPortraitMode"
            custom-class="btn-router-list"
            @click="isRouteListShowed = true" />

        <!-- 路线列表 -->
        <div class="float-route-list-panel" v-if="isRouteListShowed" :style="routeListPanelStyle">
            <RouteLineListPanel
                @choseLine="changeLine"
                @labelToggle="toggleLabel"/>
        </div>

        <!-- DETAIL INFO -->
        <RouteDetailPanel
            v-if="activeLineObj && (!store.isInPortraitMode || !isRouteListShowed)"
            :line="activeLineObj"
            :drivingInfo="drivingInfo"
            @openInGaodeApp="openInGaodeApp"
        />

        <!-- 地图 -->
        <div id="container" :style="`height: ${store.windowInsets.height}px`"></div>


    </div>
</template>

<script lang="ts" setup>
import AMapLoader from '@amap/amap-jsapi-loader'
import ICON from "@/assets/icons"
import RouteDetailPanel from "./components/RouteDetailPanel.vue"
import FloatingButton from "@/layout/FloatingButton.vue"

import {Base64} from "js-base64"
import RouteLineListPanel from "@/page/route/components/RouteLineListPanel.vue";
import axios from "axios";
import {useProjectStore} from "@/store.ts";
import {onMounted, onUnmounted, ref, watch, computed} from "vue";
import {useRoute, useRouter} from "vue-router";
import {key_service, key_web_js} from "@/mapConfig.ts";
import {EntityRoute, EntityRoutePoint} from "@/page/route/Route.ts";
import routeApi from "@/api/routeApi.ts";
import {generateMarkerContent} from "@/page/MyMapLib.ts";

const MY_POSITION = [117.129533, 36.685668]

let AMap = null
let currentDragRouting = null  // 当前导航路线

// 添加 AMap 类型定义
declare global {
    interface Window {
        map: any;
    }
}

const store = useProjectStore()
const route = useRoute()
const router = useRouter()

const isLoading = ref(false)
const activeLineObj = ref<EntityRoute | null>(null) // 当前路线对象

const isMarkerShowed = ref(true)
const currentMarkers = ref<any[]>([]) // 地图上的标记点

// 驾驶信息
const drivingInfo = ref({
    distance: '',
    time: ''
})

const routeListPanelStyle = computed(() => {
    if (store.isInPortraitMode) {
        return {
            top: 0,
            height: 'auto',
            minHeight: 'auto',
            width: '100%',
        }
    }
    return {
        height: `${store.windowInsets.height - 40}px`,
    }
})

// 跟踪天气请求以便清理
const weatherRequests = ref<any[]>([])
const isComponentMounted = ref(true)

onMounted(() => {
    AMapLoader
        .load({
            key: key_web_js, // 开发应用的 ID
            version: "2.0",   // 指定要加载的 JSAPI 的版本，缺省时默认为 1.4.15
            plugins: [
                'AMap.ToolBar', // 缩放按钮
                'AMap.Scale', // 比例尺
                'AMap.DragRoute', // 拖拽路线
                'AMap.Driving', // 导航
            ],
        })
        .then(mapItem => {
            AMap = mapItem
            window.map = new AMap.Map('container', {
                center: MY_POSITION,
                zoom: 11
            })
            window.map.addControl(new AMap.ToolBar())
            window.map.addControl(new AMap.Scale())
            if (route.query.lineId) {
                getLineInfo(route.query.lineId as string)
            }
        })
        .catch(e => {
            console.log(e)
        })
})

// 切换路线的标签显示
function toggleLabel(){
    currentMarkers.value.forEach(item => {
        if (isMarkerShowed.value){
            item.hide()
        } else {
            item.show()
        }
    })
    isMarkerShowed.value = !isMarkerShowed.value
}
function openInGaodeApp(){
    if (!activeLineObj.value || !activeLineObj.value.pathArray) {
        return
    }
    let originLnglat = activeLineObj.value.pathArray[0].position // [lng, lat]
    let destLnglat = activeLineObj.value.pathArray[activeLineObj.value.pathArray.length - 1].position // [lng, lat]
    window.map.plugin('AMap.Driving', () => {
        let currentDriving = new AMap.Driving({
            map: window.map,
            policy: AMap.DrivingPolicy.LEAST_TIME
        })
        currentDriving.search(
            new AMap.LngLat(...originLnglat),
            new AMap.LngLat(...destLnglat),
            function (status: string, result: any) {
                // result 即是对应的驾车导航信息，相关数据结构文档请参考
                // https://lbs.amap.com/api/javascript-api/reference/route-search#m_DrivingResult
                if (status === 'complete') {
                    currentDriving.searchOnAMAP({
                        origin:result.origin,
                        destination:result.destination
                    });
                    console.log(status, result)
                    console.log('绘制驾车路线完成')
                } else {
                    console.log('获取驾车数据失败')
                }
            });
    })
}


// 切换路线
function changeLine(lineId: number){
    router.push({
        name: 'RouteLine',
        query: {
            lineId
        }
    })
    if (innerWidth < 500){
        isRouteListShowed.value = false
    }
}

function getLineInfo(lineId: string) {
    routeApi
        .detail({
            id: lineId
        })
        .then(res => {
            activeLineObj.value = res.data
            if (activeLineObj.value && activeLineObj.value.paths) {
                activeLineObj.value.pathArray = JSON.parse(Base64.decode(activeLineObj.value.paths))
                loadLine(window.map, activeLineObj.value)
                loadLineLabels(window.map, activeLineObj.value)
            }
        })
}

function resizeMap() {
    let mapContainer = document.getElementById('container')!
    mapContainer.style.height = window.innerHeight + "px"
    mapContainer.style.width = window.innerWidth + "px"
}

// 载入路线信息
function loadLine(map: any, line: EntityRoute) {
    // 清除现有标记点
    currentMarkers.value.forEach(marker => {
        map.remove(marker)
    })
    currentMarkers.value = []
    isMarkerShowed.value = true

    // 切换线路之前如果存在路线，销毁已存在的路线
    if (currentDragRouting) {
        currentDragRouting.destroy()
        currentDragRouting = null
    }

    if (!line.pathArray || line.pathArray.length === 0) {
        console.warn('路线数据为空:', line)
        return
    }

    map.plugin('AMap.DragRoute', () => {
        // path 是驾车导航的起、途径和终点，官方建议最多放置 16个 途经点，以保证良好体验
        let path = line.pathArray!.map (item => item.position)
        currentDragRouting = new AMap.DragRoute(map, path, line.policy, {
            startMarkerOptions: {
                offset: new AMap.Pixel(-13, -43),
                icon: new AMap.Icon({ // 设置途经点的图标
                    size: new AMap.Size(26, 43),
                    image: ICON.start,
                    // imageOffset: new AMap.Pixel(0,0), // 图片的偏移量，在大图中取小图的时候有用
                    imageSize: new AMap.Size(26, 43) // 指定图标的大小，可以压缩图片

                }),
            },
            endMarkerOptions: {
                offset: new AMap.Pixel(-13, -43),
                icon: new AMap.Icon({ // 设置途经点的图标
                    size: new AMap.Size(26, 43),
                    image: ICON.end,
                    // imageOffset: new AMap.Pixel(0,0), // 图片的偏移量，在大图中取小图的时候有用
                    imageSize: new AMap.Size(26, 43) // 指定图标的大小，可以压缩图片

                }),
            },
            midMarkerOptions: {
                offset: new AMap.Pixel(-9, -9),
                icon: new AMap.Icon({ // 设置途经点的图标
                    size: new AMap.Size(30, 30),
                    image: ICON.midIcon,
                    // imageOffset: new AMap.Pixel(0,0), // 图片的偏移量，在大图中取小图的时候有用
                    imageSize: new AMap.Size(18, 18) // 指定图标的大小，可以压缩图片

                }),
            },
        })

        // 路线规划完成时
        currentDragRouting.on('complete', (res: any) => {
            // 路线规划完成后，返回的路线数据：设置距离、行驶时间
            let lineData = res.data.routes[0]
            let distance =  (lineData.distance / 1000).toFixed(1) // m -> km
            let time = (lineData.time / 60).toFixed() // second -> min

            fetchWeatherFromRoute(lineData.steps)

            if (activeLineObj.value) {
                activeLineObj.value.distance = parseFloat(distance)
                activeLineObj.value.time = parseInt(time)
            }

            drivingInfo.value.distance = distance
            drivingInfo.value.time = time
        })

        // 查询导航路径并开启拖拽导航
        currentDragRouting.search()
    })
}

function fetchWeatherFromRoute(steps: any){
    // 如果组件正在卸载，不获取天气
    if (!isComponentMounted.value) {
        return
    }

    // 取消任何现有的天气请求
    weatherRequests.value.forEach(request => {
        if (request && request.cancel) {
            request.cancel()
        }
    })
    weatherRequests.value = []

    let districtsMap = new Map()
    steps.forEach((item: any) => {
        item.cities && item.cities.forEach((city: any) => {
            city.districts.forEach((district: any) => {
                if (districtsMap.has(district.adcode)){

                } else {
                    districtsMap.set(district.adcode, {name: [city.name, district.name], adcode: district.adcode})
                }
            })
        })
    })

    let weatherRequestArray: any[] = []
    districtsMap.forEach((value,key) => {
        const request = getWeather(key)
        weatherRequests.value.push(request)
        weatherRequestArray.push(request)
    })

    axios
        .all(weatherRequestArray)
        .then(response => {
            // 检查组件是否仍然挂载后再更新
            if (!isComponentMounted.value) {
                return
            }
            let weatherString = '\n\n### 途经地天气信息\n\n'
            response.forEach((res, index) => {
                let weatherData = res.data.lives[0]
                weatherString = weatherString.concat(`${index + 1}. ${weatherData.province}-${weatherData.city}: ${weatherData.temperature}℃ | ${weatherData.humidity}%RH, ${weatherData.weather}\n`)
            })
            if (activeLineObj.value) {
                activeLineObj.value.note = activeLineObj.value.note.concat(weatherString)
            }
        })
        .catch(error => {
            console.log('天气请求失败:', error)
        })
}

function getWeather(adcode: string){
    return axios({
        url: 'https://restapi.amap.com/v3/weather/weatherInfo',
        params: {
            key: key_service,
            city: adcode
        }
    })
}

// 添加路线标记
function loadLineLabels(map: any, line: EntityRoute) {
    if (!line.pathArray) {
        return
    }
    line.pathArray.forEach((item, index) => {
        addMarker(map, item, index)
    })
}
function addMarker(map: any, item: EntityRoutePoint, index: number) {
    let marker = new AMap.Marker({
        position: item.position,
        title: item.note,
        draggable: false,
        content: generateMarkerContent(item.name, item.note, item.img, item.type, index),
    })
    currentMarkers.value.push(marker)
    map.add(marker)
}

// 浮动路线列表
const isRouteListShowed = ref(true) // 路线列表是否显示


watch(()=>route.query.lineId, newValue => {
    // 清除现有标记点
    currentMarkers.value.forEach(marker => {
        window.map.remove(marker)
    })
    currentMarkers.value = []

    currentDragRouting && currentDragRouting.destroy() // 销毁行程规划
    window.map.clearInfoWindow() // 清除地图上的信息窗体
    window.map.clearMap() // 删除所有标记点

    if (newValue) {
        getLineInfo(newValue as string)
    }
})

onUnmounted(() => {
    // 设置标志以防止新操作
    isComponentMounted.value = false

    // 取消任何待处理的天气请求
    weatherRequests.value.forEach(request => {
        if (request && request.cancel) {
            request.cancel()
        }
    })
    weatherRequests.value = []

    // 清除标记点
    currentMarkers.value.forEach(marker => {
        if (window.map && marker) {
            window.map.remove(marker)
        }
    })
    currentMarkers.value = []

    currentDragRouting && currentDragRouting.destroy() // 销毁行程规划
    if (window.map) {
        window.map.clearInfoWindow() // 清除地图上的信息窗体
        window.map.clearMap() // 删除所有标记点
        window.map.destroy() // 销毁地图，释放内存
    }
    window.map = null
})
</script>

<style lang="scss" scoped>
@import "../../scss/plugin";


.map-container {
    position: relative
}

.float-route-list-panel{
    min-height: 300px;
    position: absolute;
    z-index: 1000;
    top: 20px;
    left: 20px;
}

@media (max-width: $screen-width-threshold) {
    .float-route-list-panel{
        left: 50%;
        transform: translateX(-50%);
    }
}

</style>
