<template>
    <div class="imageloop_wrapper">
        <button class="arrow_ arrow_left" @click.stop="throttlePrev">
            <span class="arrow_left_content"></span>
        </button>

        <button class="arrow_ arrow_right" @click.stop="throttleNext">
            <span class="arrow_right_content"></span>
        </button>
        <slot>
            <div v-if="type === 'pic'" class="imageloop_item_wrapper">
                <myimage
                    class="imageloopItem"
                    v-for="item in itemlist"
                    :key="item.src"
                    :src="item.src"
                    :fit="item.fit"
                    :position="item.position"
                    :width="item.width"
                    :height="item.height"
                    v-show="itemlist[activeIndex].src === item.src"
                ></myimage>
            </div>
            <div v-else-if="type === 'link-pic'" class="imageloop_item_wrapper">
                <a
                    v-for="item in itemlist"
                    :key="item.src"
                    :href="item.link"
                    v-show="itemlist[activeIndex].src === item.src"
                >
                    <myimage
                        class="imageloopItem"
                        :src="item.src"
                        :fit="item.fit"
                        :position="item.position"
                        :width="item.width"
                        :height="item.height"
                    ></myimage>
                </a>
            </div>
        </slot>
    </div>
</template>

<script>
import * as lodash from "lodash";
import myimage from "./myimage.vue";
export default {
    name: "loop",
    components: { myimage },
    data() {
        return {
            activeIndex: 0,
            timer: null,
        };
    },
    inheritAttrs: false,
    props: [
        "itemlist",
        "type",
        "interVal",
    ] /* imglist是一个数组，每个item是一个对象：{width,height,position,fit,src},type是类型 */,
    methods: {
        setActiveIndex(index) {
            index = Number(index);
            if (!isNaN(index) && index === Math.floor(index)) {
                if( index < 0 ){
                    this.activeIndex = this.itemlist.length + index;
                }
                else if( index >= this.itemlist.length ){
                    this.activeIndex = index - this.itemlist.length;
                }
                else{
                    this.activeIndex = index;
                }
            }
        },
        showItem() {
            this.setActiveIndex(this.activeIndex+1);
        },
        startTimer() {
            let interTime = Number(this.interVal);
            this.timer = setInterval(this.showItem,interTime);
        },
        clearTimer(){
            if(this.timer){
                clearInterval(this.timer);
                this.timer = null;
            }
        },
        resetTimer(){
            this.clearTimer();
            this.startTimer();
        },
        prev() {
           this.setActiveIndex(this.activeIndex-1); 
           this.resetTimer();
        },
        next() {
            this.setActiveIndex(this.activeIndex+1);
            this.resetTimer();
        },
    },
    created(){
        this.throttlePrev = lodash.throttle(this.prev,300,{leading:true});
        this.throttleNext  = lodash.throttle(this.next,300,{leading:true });
    },
    mounted(){
        this.startTimer();
    }
};
</script>

<style>
.imageloop_wrapper {
    position: relative;
    overflow: hidden;
    width:100%;
    height: 100%;
}
.imageloop_item_wrapper{
    overflow: hidden;
    width: 100%;
    height:100%;
}
.imageloopItem{
    width: 100%;
    height: 100%;
}
.arrow_ {
    position: absolute;
    top: 50%;
    transform: translate(0, -50%);
    z-index: 2022;
    background-color: rgba(31, 45, 61, 0.11);
    border: none;
    padding: 0;
    margin: 0;
    border-radius: 50%;
    width: 36px;
    height: 36px;
    color: #ffffff;
    cursor: pointer;
}
.arrow_left {
    left: 15px;
}
.arrow_left_content::before {
    content: "<";
}
.arrow_right {
    right: 15px;
}
.arrow_right_content::before {
    content: ">";
}
</style>