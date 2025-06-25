<template>
    <div class="map-container">
        <div class="button-float btn-router-list"
             @click="isPointerListShowed = true"
             v-if="!isPointerListShowed && store.isInPortraitMode">
            <i class="el-icon-tickets"></i>
        </div>

        <!-- 点图列表 -->
        <div class="float-route-list-panel" v-loading="isLoading" v-if="isPointerListShowed">
            <PointerListPanel
                @chosePointer="changePointer"
                :pointerList="pointerList"/>
        </div>

        <!-- 地图 -->
        <div id="container" :style="`height: ${store.windowInsets.height}px`"></div>

        <!-- DETAIL INFO -->
        <PointerDetailPanel
            v-if="activePointerObj && (!store.isInPortraitMode || !isPointerListShowed)"
            :pointer="activePointerObj"
            @openInGaodeApp="openInGaodeApp"
        />
    </div>
</template>

<script lang="ts" setup>
import AMapLoader from '@amap/amap-jsapi-loader'
import PointerDetailPanel from "./components/PointerDetailPanel.vue"
import {key_web_js} from "@/mapConfig";
import pointerApi from "@/api/pointerApi";

import {Base64} from "js-base64"
import PointerListPanel from "./components/PointerListPanel.vue";
import {useProjectStore} from "@/pinia";
import {dateFormatter} from "@/utility";
import {EntityPointer, EntityPointerPoint} from "@/page/pointer/Pointer.ts";
import {generateMarkerContent, getMaxBoundsPointer} from "@/page/MyMapLib.ts";
import {onMounted, onUnmounted, ref, watch} from "vue";
import {useRoute, useRouter} from "vue-router";

// 添加 AMap 类型定义
declare global {
    interface Window {
        map: any;
    }
}

const store = useProjectStore()
const route = useRoute()
const router = useRouter()

const MY_POSITION = [117.129533, 36.685668]
let AMap: any = null
let cluster: any = null  // 点聚合的对象

const isLoading = ref(false)
const activePointerObj = ref<EntityPointer | null>(null) // 当前点图对象
const pointerList = ref<Array<EntityPointer>>([]) // 点图数组
const isPointerListShowed = ref(true) // 点图列表是否显示

// 跟踪标记点以便正确清理
const currentMarkers = ref<any[]>([])

// 分页器
const pager = ref({
    pageSize: 20,
    pageNo: 1,
    total: 0
})

onMounted(() => {
    AMapLoader
        .load({
            key: key_web_js, // 开发应用的 ID
            version: "2.0",   // 指定要加载的 JSAPI 的版本，缺省时默认为 1.4.15
            plugins: [
                'AMap.ToolBar', // 缩放按钮
                'AMap.Scale', // 比例尺
                'AMap.DragRoute', // 拖拽点图
                'AMap.Driving', // 导航
                'AMap.MarkerCluster', // 点聚合
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

            if (route.query.pointerId){
                getPointerInfo(route.query.pointerId as string)
            }
            getPointerList()
        })
        .catch(e => {
            console.log(e)
        })
})

function openInGaodeApp(){
    if (!activePointerObj.value || !activePointerObj.value.pointer_array) {
        return
    }
    let originLnglat = activePointerObj.value.pointer_array[0].position // [lng, lat]
    let destLnglat = activePointerObj.value.pointer_array[activePointerObj.value.pointer_array.length - 1].position // [lng, lat]
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
                    console.log('绘制驾车点图完成')
                } else {
                    console.log('获取驾车数据失败')
                }
            });
    })
}

// 获取点图列表
function getPointerList() {
    isLoading.value = true
    let requestData = {
        pageNo: pager.value.pageNo,
        pageSize: pager.value.pageSize
    }
    pointerApi
        .list(requestData)
        .then(res => {
            isLoading.value = false
            pager.value = res.data.pager
            pointerList.value = res.data.list.map((item: any) => {
                try  {
                    item.pointers = Base64.decode(item.pointers) || '[]'
                } catch (err) {
                    item.pointers = '[]'
                    console.log(err)
                }
                item.pointer_array = JSON.parse(item.pointers)
                item.date_init = dateFormatter(new Date(item.date_init))
                item.date_modify = dateFormatter(new Date(item.date_modify))
                return item
            })
        })
        .catch(err => {
            isLoading.value = false
        })
}

// 切换点图
function changePointer(pointerId: number){
    router.push({
        name: 'PointerViewer',
        query: {
            pointerId
        }
    })
    if (innerWidth < 500){
        isPointerListShowed.value = false
    }
}

function getPointerInfo(pointerId: string) {
    pointerApi
        .detail({
            id: pointerId
        })
        .then(res => {
            activePointerObj.value = res.data
            if (activePointerObj.value && activePointerObj.value.pointers) {
                activePointerObj.value.pointer_array = JSON.parse(Base64.decode(activePointerObj.value.pointers))
                loadPointerLabels(window.map, activePointerObj.value)
            }
        })
}

// 设置地图中心点：用户坐标
function setMapCenterToUserLocation(status: string, res: any) {
    if (status === 'complete') {
        let center: [number, number] = [res.position.lng, res.position.lat]
        window.map.setCenter(center)
        addMarker(window.map, {
            position: center,
            name: '我的位置',
            note: '',
            img: '',
            type: ''
        }, 0)
    } else {
        console.log(res)
    }
}

function resizeMap() {
    let mapContainer = document.getElementById('container')
    if (mapContainer) {
        mapContainer.style.height = window.innerHeight + "px"
        mapContainer.style.width = window.innerWidth + "px"
    }
}


// 添加点图标记
function loadPointerLabels(map: any, pointer: EntityPointer) {
    // 清除现有标记点和聚合
    clearAllMarkers(map)
    
    if (!pointer.pointer_array || pointer.pointer_array.length === 0) {
        return
    }
    
    let pointers = pointer.pointer_array.map((item: any) => {
                                            item.weight = 1
                                            item.lnglat = item.position
                                            return item
                                        })
    let count = pointers.length

    // 最终聚合之后，只显示一个点，这个点就是权重比较大的点，context 数据里的 clusterData 也只显示这个聚合位置的点，
    // 如果不显示报警的点，就无法获取这个点的报警状态，就无法显示报警
    // 所以增大报警点的权重
    const _renderClusterMarker = function (context: any) {
        // console.log('context cluster: ', context)
        let factor = Math.pow(context.count / count, 1 / 18);
        let div = document.createElement('div');
        let Hue = 180 - factor * 180;
        let bgColor = 'hsla(' + Hue + ',100%,100%,1)';
        let fontColor = 'hsla(' + Hue + ',100%,50%,1)';
        let borderColor = 'hsla(' + Hue + ',100%,0%,1)';
        let shadowColor = 'hsla(' + Hue + ',100%,10%,0.3)';
        div.style.backgroundColor = bgColor;
        let size = Math.round(30 + Math.pow(context.count / count, 1 / 5) * 20);
        div.style.width = div.style.height = size + 'px';
        div.style.border = `solid 1px ${borderColor}`;
        div.style.borderRadius = size / 2 + 'px';
        div.style.boxShadow = `2px 2px 5px ${shadowColor}`;
        div.innerHTML = context.count;
        div.style.lineHeight = size + 'px';
        div.style.color = fontColor;
        div.style.fontSize = '18px';
        div.style.fontWeight = 'bold';
        div.style.textAlign = 'center';
        context.marker.setOffset(new AMap.Pixel(-size / 2, -size / 2));
        context.marker.setContent(div)
    };

    const _renderMarker = function(context: any) {
        // console.log('context normal: ', context)
        let item = context.data[0] as EntityPointerPoint
        context.marker.setContent(generateMarkerContent(item.name, item.note, item.img, item.type))
        let offset = new AMap.Pixel(-9, -9);
        context.marker.setOffset(offset)
    }

    if (cluster){
        cluster.setData(pointers)
    } else {
        cluster = new AMap.MarkerCluster(
            map,       // 地图实例
            pointers,  // 海量点数据，数据中需包含经纬度信息字段 lnglat
            {
                gridSize: 30,
                renderClusterMarker: _renderClusterMarker, // 自定义聚合点样式
                renderMarker: _renderMarker, // 自定义非聚合点样式
            }
        )
    }

    let maxLocations =  getMaxBoundsPointer(pointer.pointer_array.map((item: any) => item.position))
    // 取区间的 1/4 作为地图的边界
    let lngGap = (maxLocations.max[0] - maxLocations.min[0]) / 4
    let latGap = (maxLocations.max[1] - maxLocations.min[1]) / 4

    // 新的区域极点坐标
    let min = new AMap.LngLat(maxLocations.min[0] - lngGap, maxLocations.min[1] - latGap)
    let max = new AMap.LngLat(maxLocations.max[0] + lngGap, maxLocations.max[1] + latGap)

    // 1. 多个点时，设置边界
    if (pointer.pointer_array.length > 1){
        let bounds = new AMap.Bounds(min, max)
        map.setBounds(bounds)
    }
    // 2. 一个点时，将其作为中心点
    else if (pointer.pointer_array.length === 1){
        console.log(pointer.pointer_array)
        let centerLngLat = new AMap.LngLat(...pointer.pointer_array[0].position)
        map.setCenter(centerLngLat)  // 设置地图中心点坐标
    }
    // 3. 没有点时不做处理
    else {

    }
    /*
                pointer.pointer_array.forEach((item, index) => {
                    addMarker(map, item, index)
                })*/
}

function addMarker(map: any, item: EntityPointerPoint, index: number) {
    let marker = new AMap.Marker({
        position: item.position,
        draggable: true,
        content: generateMarkerContent(item.name, item.note, item.img, item.type, index),
    })
    currentMarkers.value.push(marker) // 跟踪标记点以便清理
    map.add(marker)
}

watch(() => route.query.pointerId, newValue => {
    clearAllMarkers(window.map) // 清除地图上的所有标记和聚合
    if (newValue) {
        getPointerInfo(newValue as string)
    }
})

onUnmounted(() => {
    // 清除地图和数组中的所有标记点
    currentMarkers.value.forEach(marker => {
        if (window.map && marker) {
            window.map.remove(marker)
        }
    })
    currentMarkers.value = []
    
    // 清除聚合对象（如果存在）
    if (cluster) {
        cluster.setData([])
        cluster = null
    }
    
    if (window.map) {
        window.map.clearInfoWindow() // 清除地图上的信息窗体
        window.map.clearMap() // 删除所有标记点
    }
})

// 综合清理函数，清除所有标记点和地图元素
function clearAllMarkers(map: any) {
    // 清除我们跟踪的标记点
    currentMarkers.value.forEach(marker => {
        if (marker && map) {
            map.remove(marker)
        }
    })
    currentMarkers.value = []
    
    // 清除聚合对象（如果存在）
    if (cluster) {
        cluster.setData([])
        cluster = null
    }
    
    // 清除地图上的所有覆盖物（包括标记点、折线等）
    if (map) {
        map.clearMap()
        map.clearInfoWindow()
    }
}
</script>

<style lang="scss" scoped>
@import "../../scss/plugin";

.btn-router-list{
    position: absolute;
    top: 20px;
    left: 280px;
}

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
    .btn-router-list{
        left: auto;
        top: 10px;
        right: 10px;
    }
}

</style>
