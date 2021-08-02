<template>
    <div class="image_wrapper">
        <slot v-if="loading" name="loading">
            <div class="myimage_loading" :style="wrapper_style"><div class="myimage_loading_text" title="图片正在加载">图片正在加载</div></div>
        </slot>
        <slot v-else-if="error" name="error">
            <div class="myimage_error" :style="wrapper_style"><div class="myimage_error_text" title="图片加载失败">图片加载失败</div></div>
        </slot>
       <img v-else
       class="myimage_inner"
       :style="imagestyle"
       :src="src"> 
    </div>
</template>

<script>
const Objectfit=['none','fill','contain','cover','scale-down'];
export default {
    name:"Myimg",
    inheritAttrs:false,
    data(){
        return {
            loading:true,
            error:false,
            //imagewidth:0,
            //imageheight:0,
        }
    },
    props:['src','fit','position','width','height'],
    computed:{
        imagestyle(){
            let StyleObject = Objectfit.indexOf(this.fit)>-1?{'object-fit':this.fit}:{};
            if(this.position){
                StyleObject['object-position'] = this.position;
            }
            if(this.width){
                StyleObject['width']=this.width;
            }
            if(this.height){
                StyleObject['height']=this.height;
            }
            return StyleObject;
        },
        wrapper_style(){
            let StyleObject={};
            if(this.width){
                console.log(this.width);
                StyleObject['width']= this.width;
            }
            if(this.height){
                console.log(this.height);
                StyleObject['height']=this.height;
            }
            return StyleObject;
        },
    },
    watch:{
        src(){
            this.loadImg();
        }
    },
    mounted(){
        if(this.src){
            this.loadImg();
        }else{
            this.handleLoading();
        }

    },
    methods:{
        loadImg(){
            this.loading = true;
            this.error = false;
            let Img = new Image();
            Img.onload = this.handleLoading;
            Img.onerror = this.handleError;
            Object.keys(this.$attrs).forEach((key) => {
                const value = this.$attrs[key];
                Img.setAttribute(key, value);
            });
            Img.src = this.src;
        },
        handleLoading(){
            this.loading = false;
            this.error = false;
        },
        handleError(){
            console.log("in handleError")
            this.loading = false;
            this.error = true;
        }
    }
}
</script>

<style>
.myimamge_inner,.myimage_loading,.myimage_error{
    width: 100%;
    height: 100%;
}
.myimage_loading,.myimage_error{
    color: #abadb2;
    font-size: 15px;
    display: flex;
    justify-content: center;
    align-items: center;
    background-color: #f5f7fa;
    width: 300px;
    height: 200px;
}
.image_wrapper{
    overflow: hidden;
    margin: 3px;
    display: inline-block;
}
.myimage_loading_text,.myimage_error_text{
    overflow: hidden;
    text-overflow: ellipsis;
    word-break: keep-all;
}
</style>