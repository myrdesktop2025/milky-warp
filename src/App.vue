<template>
    <MZoom :screenshot-path="screenshotPath" :is-active="isWindowDisplayed" :class="{hide: holdHide}"/>
</template>

<script setup lang="ts">
import MZoom from "./components/Zoom.vue";
import { register } from "@tauri-apps/api/globalShortcut";
import { window } from "@tauri-apps/api";
import { convertFileSrc, invoke } from "@tauri-apps/api/tauri";
import { relaunch } from "@tauri-apps/api/process";
import { nextTick, onMounted, ref, watch } from "vue";
import { config } from "./config";
import { useMagicKeys } from "@vueuse/core";

const keys = useMagicKeys();
const holdShortcut = keys[config.holdShortcut];

const isWindowDisplayed = ref(false);
const screenshotPath = ref("");

const holdHide = ref(false);

onMounted(async () => {
    screenshotPath.value = convertFileSrc((await invoke("get_screenshot_path", {})).replaceAll("\\", "/"));
});

// Alt+C：切换显示/隐藏
register(config.shortcut, async () => {
    isWindowDisplayed.value = !isWindowDisplayed.value;

    await nextTick();

    if (!isWindowDisplayed.value) {
        await window.getCurrent().hide();
    }
    else {
        holdHide.value = false;
        screenshotPath.value = convertFileSrc((await invoke("update_screenshot", {})).replaceAll("\\", "/")) + "?" + Date.now();
        await window.getCurrent().show();
    }
});

// Shift+Alt+C：重启进程
register(config.restartShortcut, async () => {
    await relaunch();
});

// Shift+Ctrl+A：按住显示，松开隐藏
watch(holdShortcut, async (value) => {
    if (value) {
        if (isWindowDisplayed.value) {
            return
        }

        isWindowDisplayed.value = true;
        await nextTick();
        screenshotPath.value = convertFileSrc((await invoke("update_screenshot", {})).replaceAll("\\", "/")) + "?" + Date.now();
        holdHide.value = false;
    }
    else {
        holdHide.value = true;
        isWindowDisplayed.value = false;
    }
});
</script>

<style lang="scss" scoped>
.hide {
    display: none;
}
</style>

<style lang="scss">
html, body {
    margin: 0;
    background-color: transparent;

    #app {
        height: 100vh;
        overflow-y: auto;
    }
}
</style>
