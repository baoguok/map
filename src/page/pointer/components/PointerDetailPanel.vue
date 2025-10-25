<template>
    <DetailPanel
        :title="pointer.name"
        :videoLink="pointer.video_link"
        :infoArray="infoArray"
        :markdownNote="pointer.note"
        :isShowQr="false"
    >
        <template #footer>
            <div v-if="store.isAdmin || (store.authorization && store.authorization.uid) === pointer.uid" >
                <ElButton
                    v-if="pointer.id !== undefined"
                    plain size="small"
                    @click="goToEditCurrentPointer"
                    icon="EditPen">编辑</ElButton>
            </div>
        </template>
    </DetailPanel>
</template>

<script lang="ts" setup>
import {useProjectStore} from "@/store.ts";
import {EntityPointer} from "@/page/pointer/Pointer.ts";
import {computed} from "vue";
import {useRouter} from "vue-router";
import DetailPanel from "@/layout/DetailPanel.vue";
import { dateFormatter } from "@/utility";

const store = useProjectStore()
const router = useRouter()

const props = withDefaults(defineProps<{
    pointer: EntityPointer
}>(), {
    pointer: {
        name: '', // *点图名
        area: '', // *地域
        video_link: '', // 路径视频演示
        pointers: [], // *路径点
        note: '', // 备注
        thumb_up: 0, // *点赞数
        is_public: 1, // *是否公开
    }
})

function goToEditCurrentPointer(){
    router.push({
        name: 'PointerEditor',
        query: {
            pointerId: props.pointer.id
        }
    })
}

const infoArray = computed(() => {
    return [
        { title: '点图区域', value: props.pointer.area },
        { title: '创建用户', value: props.pointer.nickname },
        { title: '创建时间', value: dateFormatter(new Date(props.pointer.date_create), 'yyyy/MM/dd hh:mm:ss')},
    ]
})
</script>

<style lang="scss" scoped>
@import "../../../scss/plugin";
</style>
