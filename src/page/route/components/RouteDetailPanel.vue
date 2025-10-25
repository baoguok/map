<template>
    <DetailPanel
        :title="line.name"
        :videoLink="line.video_link"
        :infoArray="infoArray"
        :markdownNote="line.note"
        :isShowQr="false"
    >
        <template #footer>
            <div v-if="store.isAdmin || (store.authorization && store.authorization.uid) === line.uid" >
                <ElButton
                    v-if="line.id !== undefined"
                    plain size="small"
                    @click="goToEditCurrentLine"
                    icon="EditPen">编辑</ElButton>
            </div>
        </template>
    </DetailPanel>
</template>

<script lang="ts" setup>
import {policyMap} from "@/page/route/DrivingPolicy"
import {useProjectStore} from "@/store.ts";
import {useRouter} from "vue-router";
import {EntityRoute} from "@/page/route/Route.ts";
import { dateFormatter } from "@/utility";
import { computed } from "vue";
import DetailPanel from "@/layout/DetailPanel.vue";

const props = defineProps<{
    line: EntityRoute,
    drivingInfo?: {}
}>()

const store = useProjectStore()
const router = useRouter()

function goToEditCurrentLine(){
    router.push({
        name: 'RouteEditor',
        query: {
            lineId: props.line.id
        }
    })
}

/**
 * 路线信息列表
 */
const infoArray = computed(() => {
    let tempArray = [
        {title: '路线区域', value: props.line.area},
        {title: '公开状态', value: props.line.is_public? '公开': '私有'},
    ]

    if (props.line.pathArray && props.line.pathArray.length > 0) {
        tempArray.push({title: '节点数量', value: props.line.pathArray.length + ' 个'})
    }

    if (props.line.policy !== undefined) {
        tempArray.push({title: '路线策略', value: policyMap.get(props.line.policy)})
    }

    if (props.line.road_type) {
        tempArray.push({title: '路面类型', value: props.line.road_type})
    }

    if (props.line.seasons) {
        tempArray.push({title: '推荐季节', value: props.line.seasons})
    }

    if (props.line.distance) {
        tempArray.push({title: '距离/时间', value: `${props.line.distance} km / ${props.line.time} min`})
    }

    if (props.line.nickname) {
        tempArray.push({title: '创建用户', value: props.line.nickname})
    }

    if (props.line.date_init) {
        tempArray.push({title: '创建时间', value: dateFormatter(new Date(props.line.date_init), 'yyyy/MM/dd hh:mm:ss')})
    }

    return tempArray
})

</script>

<style lang="scss" scoped>
@import "../../../scss/plugin";
</style>
