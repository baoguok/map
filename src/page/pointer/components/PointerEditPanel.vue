<script lang="ts" setup>
import ClipboardJS from 'clipboard'
import {qiniu_bucket_name, qiniu_img_base_url, thumbnail200_suffix} from "@/mapConfig.ts";
import * as qiniu from "qiniu-js";
import {getAuthorization} from "@/utility.ts";
import {ElMessage} from "element-plus";
import {getUploadToken} from "@/api/fileApi.ts";
import {nextTick, onMounted, onUnmounted, ref, watch} from "vue";
import {EntityPointerPoint, EnumPointerType, EnumPointerTypeMap} from "@/page/pointer/Pointer.ts";
import {useProjectStore} from "@/pinia.ts";

const pointerName = ref('') // 当前点的地名
const pointerNote = ref('') // 标记note
const pointerType = ref('') // 标记类别
const pointerImg = ref('') // 标记图片地址
const avatarFile = ref()

const projectStore = useProjectStore()

const clipboardPointerData = ref('') // 要复制的所有路线点的数据
let clipboard = null

const imgSuffix = thumbnail200_suffix
const currentPointIndex = ref(null) // 当前图片需要放到哪个点位上

const refInputName = ref()


const emit = defineEmits(['print', 'showPointer'])
const props = defineProps<{
    lng: number,
    lat: number,
    keyword: string,
}>()
const modelData = defineModel<Array<EntityPointerPoint>>()


onMounted(() => {
    // 1. 绑定剪贴板操作方法
    clipboard = new ClipboardJS('.lnglat', {
        text: trigger => {
            // 2. trigger 就是点击的 dom 元素，所以就可以通过它来获取到它的属性 'dataClipboard' 的值了
            // 3. 从这个方法返回的 string 就是会复制到剪贴板中的内容，所以可以复制任何内容了，也可以在这里处理一些逻辑
            // 4. 我在这里就只是单纯的输出了事先绑定好的值
            return trigger.getAttribute('data-clipboard-text')
        },
    })
    // 5. 当复制成功的时候提示复制成功
    clipboard.on('success', ()=>{  // 还可以添加监听事件，如：复制成功后提示
        ElMessage.success('复制成功')
    })
    pointerName.value = props.keyword
})

onUnmounted(()=>{
    clipboard.destroy()
})

watch(() => props.keyword, newValue => {
    pointerName.value = newValue
})


/**
 * 编辑字段
 **/
let modifyingArrayIndex = 0 // 编辑的 index
let modifyingParamName = '' // 编辑的字段名

const isModalShowing = ref(false)
const currentModifyingString = ref('')

function modifyString(str: string, index: number, paramName: string){
    modifyingParamName = paramName
    modifyingArrayIndex = index
    currentModifyingString.value = str
    isModalShowing.value =  true
}
// 提交对字符串的修改
function submitStringChange(){
    modelData.value[modifyingArrayIndex][modifyingParamName] = currentModifyingString.value
    isModalShowing.value = false
    currentModifyingString.value = ''
}
function closeModal(){
    isModalShowing.value = false
    currentModifyingString.value = ''
}


/**
 *
 * 对于 ModelData 的操作
 */

function clearAllPointers(){
    modelData.value = []
}

function reversePointers(){
    modelData.value?.reverse()
}




function uploadAvatar(event){
    if (!getAuthorization()){
        ElMessage.error('请登录后操作')
        return
    }
    if (event.target.files.length > 0){
        avatarFile.value = event.target.files[0]
        if (!/image\/.*/.test(avatarFile.value.type)){
            ElMessage.warning('请选择图片文件')
            event.target.value = '' // 清空 Input 内容
            return
        }
        if (avatarFile.value.size > 1024 * 1024 * 8){
            ElMessage.warning('图片应小于 8MB')
            event.target.value = '' // 清空 Input 内容
            return
        }

        getUploadToken({
            bucket: qiniu_bucket_name
        })
            .then(res => {
                console.log('get token success')
                // 上传文件
                const observer = {
                    next: res => {
                        console.log('next: ',res)
                    },
                    error: err => {
                        console.log(err)
                    },
                    complete: res => {
                        // res = {hash: 'hash', key_service: 'key_service'}
                        console.log('complete: ',res)
                        if (currentPointIndex.value !== null){
                            let tempData = modelData.value // 临时数组
                            tempData[currentPointIndex.value].img = qiniu_img_base_url + res.key
                            emit('setData', [...tempData])
                            currentPointIndex.value = null // 指向归位
                        } else {
                            pointerImg.value = qiniu_img_base_url + res.key
                        }
                    }
                }
                const observable = qiniu.upload(avatarFile.value, null, res.data, {}, {})
                const subscription = observable.subscribe(observer) // 上传开始
                // subscription.unsubscribe() // 上传取消
            })
            .catch(err => {
                ElMessage.error('获取上传 token 失败')
            })
    }
}
// enter 时触发的方法
function addNewPointToPointerWithKeyEnter(){
    if(isInputValidated()){
        modelData.value.push({
            position: [props.lng, props.lat],
            note: pointerNote.value,
            name: pointerName.value
        })
        pointerName.value = ''
        pointerNote.value = ''
    }
    nextTick(()=>{
        refInputName.value.focus()
    })
}
// 点击时触发的方法
function addNewPointToPointer(){
    if (isInputValidated()){
        modelData.value.push({
            position: [props.lng, props.lat],
            note: pointerNote.value,
            name: pointerName.value,
            type: EnumPointerType[pointerType.value],
            img: pointerImg.value
        })
    }
}
function isInputValidated(){
    if (!props.lng || !props.lat){
        ElMessage({
            message: '坐标未选定',
            type: 'warning'
        })
        return false
    }
    if (!pointerName.value){
        ElMessage({
            message: '地名未填写',
            type: 'warning'
        })
        refInputName.value.focus()
        return false
    }
    return true
}


function move(index: number, direction: 'up'|'down'){
    let indexExchange = direction === 'up'? index - 1 : index + 1
    let tempItem = modelData.value[index]
    let preItem = modelData.value[indexExchange]
    let tempData = modelData.value // 临时数组
    tempData[indexExchange] = tempItem
    tempData[index] = preItem
    emit('setData', [...tempData])
}
function pointerPointDelete(index){
    modelData.value.splice(index, 1)
}




watch(modelData, newValue => {
    clipboardPointerData.value = JSON.stringify(newValue)
})
</script>

<template>
    <div>
        <div class="circle-panel card" :style="`height: ${projectStore.contentInsets.heightContent - 200}px`">
            <div class="toolbar">
                <ElButton class="lnglat" :data-clipboard-text="JSON.stringify(modelData)" size="small" type="info" icon="CopyDocument">复制 JSON 数据</ElButton>
                <ElButton size="small" type="danger" @click="clearAllPointers" icon="RefreshLeft">清空</ElButton>
                <ElButton size="small" type="warning" @click="reversePointers" icon="Sort">倒序</ElButton>
            </div>

            <table class="log">
                <thead>
                <tr>
                    <td style="width: 30px;">#</td>
                    <td style="width: 80px;">经纬</td>
                    <td style="width: 100px;">地点</td>
                    <td style="width: 100px;">备注</td>
                    <td style="width: 80px;">类别</td>
                    <td style="width: 40px;">图片</td>
                    <td style="width: 80px;">操作</td>
                </tr>
                </thead>
                <tbody>
                <tr>
                    <td><i class="el-icon-aim"></i></td>
                    <td>
                        <ElTooltip effect="dark" trigger="hover" placement="left" content="点击复制坐标值">
                            <div class="lnglat" :data-clipboard-text="`[${lng}, ${lat}]`">
                                <div class="lng">经: {{lng || '--'}}</div>
                                <div class="lat">纬: {{lat || '--'}}</div>
                            </div>
                        </ElTooltip>
                    </td>
                    <td>
                        <ElInput
                            @keyup.native.enter="addNewPointToPointerWithKeyEnter"
                            clearable
                            ref="refInputName" class="input-focus" size="small"
                            placeholder="标记名"
                            v-model="pointerName"/>
                    </td>
                    <td>
                        <ElInput
                            type="textarea"
                            autosize
                            clearable
                            ref="inputNote" class="input-focus" size="small"
                            placeholder="备注"  :rows="1"
                            v-model="pointerNote"/>
                    </td>
                    <td>
                        <ElSelect size="small" v-model="pointerType">
                            <template v-for="key in EnumPointerType" :key="key">
                                <ElOption v-if="isNaN(EnumPointerType[key])" :label="EnumPointerTypeMap.get(key)" :value="key"/>
                            </template>
                        </ElSelect>
                    </td>
                    <td>
                        <ElTooltip effect="dark" trigger="hover" placement="top" content="点击上传图片">
                            <div class="img-wrapper">
                                <img v-if="pointerImg" :src="`${pointerImg}-${imgSuffix}`" alt="图片">
                                <label class="logo avatar" for="avatar">
                                    <ElIcon size="14"><Upload/></ElIcon>
                                </label>
                                <input type="file" @change="uploadAvatar" id="avatar">
                            </div>
                        </ElTooltip>

                    </td>
                    <td>
                        <ElButton size="small" type="success" @click="addNewPointToPointer" icon="Plus">添加</ElButton>
                    </td>
                </tr>

                <tr v-for="(item, index) in modelData" :key="index">
                    <td>{{index + 1}}</td>
                    <!--                <td @click="modifyString(item.position, index, 'position')">-->
                    <td>
                        <div class="lnglat" :data-clipboard-text="`[${lng}, ${lat}]`">
                            <div class="lng">lng: {{item.position[0]}}</div>
                            <div class="lat">lat: {{item.position[1]}}</div>
                        </div>
                    </td>
                    <td @click="modifyString(item.name, index, 'name')">
                        <div class="textarea">{{item.name}}</div>
                    </td>
                    <td @click="modifyString(item.note, index, 'note')">
                        <div class="textarea">{{item.note}}</div>
                    </td>
                    <td>
                        <ElSelect size="small" v-model="item.type">
                            <template v-for="key in EnumPointerType" :key="key">
                                <ElOption v-if="isNaN(EnumPointerType[key])" :label="EnumPointerTypeMap.get(key)" :value="key"/>
                            </template>
                        </ElSelect>
                    </td>
                    <td>
                        <div class="img-wrapper">
                            <img v-if="item.img" :src="`${item.img}-${imgSuffix}`" alt="图片">
                            <label class="logo avatar" for="avatar" @click="currentPointIndex = index">
                                <ElIcon size="14"><Upload/></ElIcon>
                            </label>
                        </div>
                    </td>
                    <td>
                        <div :class="['operation', {'align-items-start': index > 0}, {'align-items-end': index < modelData.length - 1}]">
                            <div class="move">
                                <ElIcon v-if="index > 0"  @click="move(index, 'up')" size="12"><CaretTop/></ElIcon>
                                <ElIcon v-if="index < modelData.length - 1" @click="move(index, 'down')" size="12"><CaretBottom/></ElIcon>
                            </div>
                            <div class="delete">
                                <ElIcon @click="pointerPointDelete(index)" size="15"><CircleClose/></ElIcon>
                            </div>
                        </div>
                    </td>
                </tr>
                </tbody>
            </table>
        </div>

        <ElDialog
            center
            title=""
            append-to-body
            v-model="isModalShowing"
            :before-close="closeModal"
            width="400px">
            <div>
                <ElForm label-position="top">
                    <ElFormItem label="修改">
                        <ElInput type="textarea" :rows="5" v-model="currentModifyingString"/>
                    </ElFormItem>
                </ElForm>
            </div>
            <template #footer class="dialog-footer">
                <ElButton type="info" @click="isModalShowing = false" icon="Close">取消</ElButton>
                <ElButton type="primary" @click="submitStringChange" icon="Check">确定</ElButton>
            </template>
        </ElDialog>
    </div>
</template>

<style scoped lang="scss">
@use 'sass:math';

@import "../../../scss/plugin";
.circle-panel {
    overflow-y: auto;
    width: 500px;
    padding: 0;
}

.align-items-start{
    align-items: flex-start;
}
.align-items-end{
    align-items: flex-end;
}

.img-wrapper{
    display: flex;
    justify-content: flex-start;
    input[type=file]{
        display: none;
    }
    img{
        height: 30px;
        display: block;
        margin-right: 5px;
    }
    label{
        display: flex;
        justify-content: flex-start;
        background-color: white;
        cursor: pointer;
        i{
            border: 1px solid $border-light;
            width: 30px;
            height: 30px;
            line-height: 30px;
            text-align: center;
            &:hover{
                border-color: $green;
            }
        }

    }
}


.toolbar{
    padding: 6px;
}

table{
    width: 100%;
}
thead{
    td {
        font-size: 0.8rem;
        color: $text-main;
        padding: 8px 2px 3px 5px;
        border-bottom: 1px solid $border-normal;
        &:last-child, &:first-child {
            text-align: center;
        }
        &:nth-child(1) {width: 10%;}
        &:nth-child(2) {width: 25%;}
        &:nth-child(3) {width: 25%;}
        &:nth-child(4) {width: 25%;}
        &:nth-child(5) {width: 15%;}
    }
}
tbody{
    tr{
        &:hover{
            background-color: transparentize($color-main, 0.9);
        }
    }
    tr:nth-child(2n + 1){
        background-color: #f2f2f2;
    }
    td{
        vertical-align: top;
        font-size: 0.7rem;
        padding: 2px 2px 2px 5px;
        &:last-child, &:first-child {
            text-align: center;
        }
    }
}

@media (max-width: $screen-width-threshold) {
    .circle-panel{
        width: 100%;
    }
}


</style>
