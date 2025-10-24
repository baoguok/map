<template>
    <ElContainer>
        <ElContainer>

            <!-- 悬浮菜单按钮 -->
            <transition
                enter-active-class="animate__bounceInUp"
                leave-active-class="animate__bounceOutDown"
            >
                <div v-show="store.isShowFloatingMenuBtn" class="menu-btn animate__animated " @click="toggleMenu">
                    <img src="../assets/logo.png" alt="logo">
                </div>
            </transition>

            <transition
                enter-active-class="animate__faster animate__slideInLeft"
                leave-active-class="animate__faster animate__slideOutRight"
            >
                <Aside class="animate__animated" v-show="!store.isShowFloatingMenuBtn"/>
            </transition>
            <ElContainer>
                <ElMain>
                    <Container>
                        <RouterView/>
                    </Container>
                </ElMain>
            </ElContainer>
        </ElContainer>
    </ElContainer>
</template>

<script lang="ts" setup>
import Aside from "./Aside.vue";
import {useProjectStore} from "@/store.ts";
import Container from "@/layout/Container.vue";

const store = useProjectStore()

function toggleMenu() {
    store.isShowFloatingMenuBtn = !store.isShowFloatingMenuBtn
}
</script>

<style lang="scss" scoped>

@import "../scss/plugin";

.el-main {
    padding: 0;
}

.menu-btn {
    @include box-shadow(1px 1px 3px rgba(0, 0, 0, 0.1));
    background-color: white;
    overflow: hidden;
    padding: 6px;
    position: fixed;
    bottom: 10px;
    left: 10px;
    height: $width-float-btn;
    width: $width-float-btn;
    @include border-radius(100px);
    z-index: 999;

    img {
        width: 100%;
        display: block;
    }

    &.active {
        transform: translateY(2px);
    }
}
</style>
