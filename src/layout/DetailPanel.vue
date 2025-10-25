<template>
    <div :class="[
            'detail',
            'card',
            {'closed': !showContent},
            {'center': store.isInPortraitMode}
        ]">
        <div class="title">
            <div class="title-name">
                {{props.title}}
            </div>
            <a v-if="props.videoLink" target="_blank" class="video-link" :href="props.videoLink">
                <ElIcon size="20"><VideoCameraFilled/></ElIcon>
            </a>
            <div class="collapse-btn">
                <ElButton size="small" @click="toggleContent">
                    <ElIcon v-if="showContent" ><ArrowUp/></ElIcon>
                    <ElIcon v-else><ArrowDown/></ElIcon>
                </ElButton>
            </div>
        </div>

        <div class="content" v-if="showContent">
            <div class="info-list" v-if="props.infoArray && props.infoArray.length > 0">
                <div class="info" v-for="info in props.infoArray" :key="info.title">
                    <p class="info-title">{{info.title}}</p><p class="info-value">{{info.value}}</p>
                </div>
            </div>


            <!-- 备注 -->
            <div class="note markdown"
                :style="`max-height: ${height}px`"
                v-if="contentHtml"
                v-html="contentHtml"/>

            <!-- 底部按钮 -->
            <div class="button-center">
                <slot name="footer"/>
            </div>

            <!-- 二维码 -->
           <div class="qr" v-if="props.isShowQr">
               <img :src="qrImg" alt="">
               <p>扫一扫，打开该页面</p>
           </div>
        </div>
    </div>
</template>

<script lang="ts" setup>
import {marked} from "marked";
import {useProjectStore} from "@/store.ts";
import { computed, onMounted, ref } from "vue";
import * as QRCode from "@/lib/qr.js"


const props = withDefaults(defineProps<{
    title: string, // 路线名称
    videoLink?: string, // 视频链接
    infoArray?: Array<{  // 信息列表
        title: string,
        value: string
    }>,
    isShowQr?: boolean,
    markdownNote: string, // 备注 markdown
}>(), {
    isShowQr: false,
})

const store = useProjectStore()

const showContent = ref(true)
const qrImg = ref('')
const height = ref(innerHeight * 0.5)

onMounted(() => {
    if (props.isShowQr) {
        // qrImg.value = QRCode.generatePNG(window.location.href)
    }
})

const contentHtml = computed(() => {
    return marked.parse(props.markdownNote)
})

function toggleContent(){
    showContent.value = !showContent.value
}


</script>

<style lang="scss" scoped>
@import "@/scss/plugin";

i{
    @extend .unselectable;
}
.detail{
    z-index: 999;
    backdrop-filter: blur(3px) saturate(120%);
    position: absolute;
    top: 20px;
    right: 20px;
    padding: 0 !important;
    width: 350px;
    .title{
        display: flex;
        justify-content: center;
        align-items: center;
        position: relative;
        text-align: center;
        padding: 10px 0 15px;
        font-size: 1rem;
        color: $text-main;
        border-bottom: 1px solid $border-normal;
        .title-name{
            padding: 0 5px;
            font-size: 1.5rem;
            font-weight: bold;
        }
        .video-link{
            display: block;
            &:hover{
                color: $color-main;
            }
            &:active{
            }
        }
        .collapse-btn{
            padding: 0 5px;
        }
    }
    &.closed{
        .title{
            padding: 5px 0 5px;
            border: none;
        }
    }
    &.center{
        left: 50%;
        transform: translateX(-50%);
    }
}

.content{
    padding-bottom: 20px;
    @include transition(all 0.3s);
    color: $text-subtitle;
    font-size: 0.8rem;
}
.info-list{
    background-color: white;
    font-size: 12px;
    padding: 10px 20px;
}
.info{
    width: 100%;
    line-height: 1.5;
    position: relative;
    display: flex;
    flex-flow: row nowrap;
    justify-content: space-between;
    p{
        z-index: 10;
        background-color: white;
    }
    &-title{
        padding-right: 10px;
        color: $text-main;
    }
    &-value{
        padding-left: 10px;
    }

    &::after{
        width: 100%;
        height: 1px;
        position: absolute;
        bottom: 50%;
        left: 0;
        content: '';
        border-bottom: 1px dashed $border-normal;
    }
}
.note{
    overflow-y: auto;
    padding: 10px 20px;
    border-top: 1px solid $border-normal;
    line-height: 1.5;
    color: $text-main;
}

.qr{
    flex-flow: column nowrap;
    display: flex;
    justify-content: center;
    align-items: center;
    padding-bottom: 15px;
    img{
        display: block;
        width: 120px;
    }
    p{
        font-size: $fz-small;
    }
}

@media (max-width: $screen-width-threshold) {
    .detail{
        &.center{
            top: 10px;
        }
    }
}
</style>
