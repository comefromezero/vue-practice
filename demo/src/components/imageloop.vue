<template>
    <div class="imageloop_wrapper">
        <button class="arrow_ arrow_left">
            <span class="arrow_left_content"></span>
        </button>


        <button class="arrow_ arrow_right">
            <span class="arrow_right_content"></span>
        </button>
        <myimage  ></myimage>
    </div>
</template>

<script>
import myimage from './myimage.vue'
export default {
    name:"loop",
    components:{myimage,},
    data(){
        return{
            curIndex:0,
        }
    },
    props:['imglist'], /* imglist是一个数组，每个item是一个对象：{width,height,position,fit,src} */
    
}
</script>

<style>
.imageloop_wrapper{
    position:relative;
}
.arrow_{
    position: absolute;
    top: 50%;
    transform: translate(0,-50%);
    z-index: 2022;
    background-color: rgba(31,45,61,0.11);
    border: none;
    padding: 0;
    margin:0;
    border-radius: 50%;
    width: 36px;
    height: 36px;
    color: #FFFFFF;
    cursor: pointer;
}
.arrow_left{
    left: 16px;
}
.arrow_left_content::before{
    content: "<";
}
.arrow_right{
    right: 16px;
}
.arrow_right_content::before{
    content: ">";
}
</style>