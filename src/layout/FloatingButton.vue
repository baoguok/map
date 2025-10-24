<template>
    <div 
        :class="['button-float', customClass]"
        @click="handleClick"
        v-if="show">
        <ElIcon> <Tickets/> </ElIcon>
    </div>
</template>

<script lang="ts" setup>

interface Props {
    show?: boolean
    customClass?: string
    clickHandler?: () => void
}

const props = withDefaults(defineProps<Props>(), {
    show: true,
    customClass: '',
})

const emit = defineEmits<{
    click: []
}>()


const handleClick = () => {
    if (props.clickHandler) {
        props.clickHandler()
    }
    emit('click')
}
</script>

<style lang="scss" scoped>
@import "../scss/plugin";

.button-float{
    position: fixed;
    bottom: 10px;
    left: 60px;
    z-index: 1000;
    background-color: white;
    width: $width-float-btn;
    font-size: 25px;
    line-height: $width-float-btn;
    text-align: center;
    height: $width-float-btn;
    @include border-radius($width-float-btn);
    border: 1px solid $border-normal;
    @extend .btn-like;
    &:hover{
        border-color: $color-primary;
        color: $color-primary;
        @include box-shadow(1px 2px 2px transparentize(black, 0.8))
    }
}
</style>
