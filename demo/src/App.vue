<template>
    <div>
        <div style="border:1px solid #abadb2;padding:4px;">
            <h3>图片组件示例</h3>
            <hr>
            <myimage
                v-for="item in imglist"
                :src="item.src"
                :key="item.src"
                :width="item.width"
                :height="item.height"
                :fit="item.fit"
            ></myimage>
            <hr>
            <button @click="ChangeSrc" type="button">更改src地址，成功加载图片</button>
        </div>
        <br>
        <div>
            <myimageloop style="width:300px;height:300px;background:red;"></myimageloop>
        </div>
    </div>
</template>

<script>
import myimage from "./components/myimage.vue"
import myimageloop from './components/imageloop.vue'
export default {
    name: "App",
    components: {
        myimage,
        myimageloop,
    },
    data() {
        return {
            imglist: [
                {
                    src: "1",
                    width: "80px",
                    height: "100px",
                },
                {
                    src: "2",
                    width: "200px",
                    height: "100px",
                    fit: "cover",
                },
                {
                    src: "3",
                    width: "200px",
                    height: "100px",
                },
                {
                    src: "4",
                    width: "200px",
                    height: "100px",
                },
            ],
        };
    },
    methods: {
        ChangeSrc() {
            this.imglist[1].src = "/test.png";
        },
    },
};
</script>

<style>
</style>