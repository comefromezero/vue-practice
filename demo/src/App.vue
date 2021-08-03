<template>
    <div>
        <myimage  v-for="item in imglist" :src="item.src" :key="item.src"  :width="item.width" :height="item.height" :fit="item.fit"></myimage>
        <button @click="ChangeSrc">按钮</button>
    </div>
</template>

<script>
import myimage from './components/myimage.vue'
export default {
    name:"App",
    components:{
        myimage,
    },
    data(){
        return {
            imglist:[
                {
                    src:"1",
                    width:"80px",
                    height:"100px"
                },
                {
                    src:"2",
                    width:"200px",
                    height:"100px",
                    fit:"cover",
                },
                {
                    src:"3",
                    width:"200px",
                    height:"100px"
                },
                {
                    src:"4",
                    width:"200px",
                    height:"100px"
                },
            ]
        }
    },
    methods: {
        ChangeSrc(){
            this.imglist[1].src="/test.png"
        },
    },
}
</script>

<style>

</style>