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
       :src="src" 
       v-bind="$attrs"
       v-on="$listeners"
       > 
        <!-- 通过v-bind和v-on将外部设定的html属性以及绑定的event回调绑定到当前img对象上。 -->
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
    props:['src','fit','position','width','height'],//width和height是CSS的width与height,position同object-position，fit同object-fit。始终指定width和height可以获得更好的用户体验。
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
            /* 创建一个Image对象，用来测试img是否可以被加载成功，从而确定html标签的img对象是否能够加载成功，然后根据成功与否选择渲染。 */
            let Img = new Image();
            Img.onload = this.handleLoading;
            Img.onerror = this.handleError;
            /* 绑定attr到当前Img对象，保持本Img对象与标签的attr一致（不包括clsss和style相关属性，因为CSS相关属性不影响加载；也不包括event回调处理，因为回调只有在加载之后才会触发。） */
            Object.keys(this.$attrs).forEach((key) => {
                const value = this.$attrs[key];
                Img.setAttribute(key, value);
            });
            console.log(this.$attrs);
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
.myimage_inner,.myimage_loading,.myimage_error{
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
   /*  width:100%;
    height:100%;
 */}
.myimage_loading_text,.myimage_error_text{
    overflow: hidden;
    text-overflow: ellipsis;
    word-break: keep-all;
}
</style>