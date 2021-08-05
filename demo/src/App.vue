<template>
    <div>
        <div style="border:1px solid #abadb2;padding:4px;">
            <h3>图片组件示例</h3>
            <h4>始终通过width和height为图片指定width和height，而不要通过style的width与height限制大小。</h4>
            <hr>
            <myimage
                v-for="item in imglist"
                :src="item.src"
                :key="item.src"
                :width="item.width"
                :height="item.height"
                :fit="item.fit"
                :position="item.position"
            ></myimage>
            <hr>
            <button @click="ChangeSrc" type="button">更改src地址，成功加载图片</button>
        </div>
        <br>
        <div style="border:1px solid #abadb2;padding:4px;">
            <h3>轮播图示例</h3>
            <hr>
            <h4>图片类型的轮播图</h4>
            <myimageloop style="width:400px;height:300px;" type="pic" interVal="4000" :itemlist="itemlist"></myimageloop>
            <hr>
            <h4>可点击类型的轮播图</h4>
            <myimageloop style="width:400px;height:300px;" type="link-pic" interVal="4000" :itemlist="itemlist"></myimageloop>
            <hr>
        </div>
        <div id="main" style="width: 600px;height:400px;"></div>
    </div>
</template>

<script>
import myimage from "./components/myimage.vue"
import myimageloop from './components/imageloop.vue'
import * as echarts from '../lib/echarts-5.1.2/dist/echarts.min'
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
                    fit: "cover",
                    width:"200px",
                    height:"150px"
                },
                {
                    src: "3",
                    width:"500px",
                    height:"300px"
                },
                {
                    src: "4",
                    width: "200px",
                    height: "100px",
                },
            ],
            itemlist: [
                {
                    src:"/1.png",
                    fit:"none",
                    link:"https://www.baidu.com"
                },
                {
                    src:"/2.png",
                    fit:"contain",
                    link:"https://www.baidu.com"
                },
                {
                    src:"/3.png",
                    fit:"fill",
                    link:"https://www.baidu.com"
                },
                {
                    src:"/4.png",
                    fit:"scale-down",
                    link:"https://www.baidu.com"
                },
                {
                    src:"/5.png",
                    fit:"cover",
                    link:"https://www.baidu.com"
                },
            ]
        };
    },
    methods: {
        ChangeSrc() {
            this.imglist[1].src = "/test.png";
        },
    },
    mounted(){
        // 基于准备好的dom，初始化echarts实例
        let myChart = echarts.init(document.getElementById('main'));

        // 指定图表的配置项和数据
        let option = {
            title: {
                text: 'ECharts 入门示例'
            },
            tooltip: {},
            legend: {
                data:['销量']
            },
            xAxis: {
                data: ["衬衫","羊毛衫","雪纺衫","裤子","高跟鞋","袜子"]
            },
            yAxis: {},
            series: [{
                name: '销量',
                type: 'bar',
                data: [5, 20, 36, 10, 10, 20]
            }]
        };

        // 使用刚指定的配置项和数据显示图表。
        myChart.setOption(option);
    },
};
</script>

<style>
</style>