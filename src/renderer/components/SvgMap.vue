<template>
    <div>
        <div class="select">
            <el-row :gutter="20">
                <el-col :span="4">&nbsp</el-col>
                <el-col :span="4">
                    <el-select v-model="dayValue" placeholder="选择日期">
                        <el-option
                                v-for="item in options1"
                                :key="item.value"
                                :label="item.label"
                                :value="item.value">
                        </el-option>
                    </el-select>
                </el-col>
                <el-col :span="4">
                    <el-select v-model="timePointValue" :disabled="disableTimeSelect" filterable placeholder="选择时刻">
                        <el-option
                                v-for="item in options2"
                                :key="item.value"
                                :label="item.label"
                                :value="item.value">
                        </el-option>
                    </el-select>
                </el-col>
                <el-col :span="4">
                    <!--<el-input placeholder="人员ID"></el-input>-->
                    <el-select v-model="personValue" :disabled="disableIdSelect"
                               multiple filterable remote reserve-keyword :remote-method="remoteMethod"
                               :loading="loading" placeholder="请选择人员id">
                        <el-option
                                v-for="item in personIdOptions"
                                :key="item.value"
                                :label="item.label"
                                :value="item.value">
                        </el-option>
                    </el-select>
                </el-col>
                <el-col :span="4">
                    <el-switch
                            v-model="heatMapSwitch"
                            active-color="#F5D368"
                            inactive-color="#49AFFF"
                            active-text="热力图"
                            inactive-text="路径图"
                            style="position: relative; top: 8px;"
                    ></el-switch>
                </el-col>
                <el-col :span="4">&nbsp;</el-col>
            </el-row>


            <el-dialog title="传感器人数统计" :visible.sync="singleSensorTableVisible">
                传感器：{{sensorId}} <br>
                总人数:{{countPersons}}
                <el-table :data="idList">
                    <el-table-column prop="id" label="人员ID"></el-table-column>
                    <el-table-column prop="time" label="时间"></el-table-column>
                </el-table>
            </el-dialog>

            <el-dialog title="范围人数统计" :visible.sync="rangeSensorFigureVisible" width="65%">
                <!--                位置：{{rangeName}}-->
                <div id="rangeSensorFigure" style="width: 1200px; height: 500px; margin-top: 10px"></div>
            </el-dialog>
        </div>
        <div class="svgContainer">
        <svg></svg>
        </div>
    </div>
</template>

<script>
    import echarts from "echarts";

    let d3 = require('d3');
    let db = require('../database');
    let util = require('../util');
    import StaticData from '../static data';

    function renderFloorMap(floor) {
        d3.select('svg')
            .selectAll('.floor')   // 显示方块
            .data(floor)
            .enter()
            .append('rect')
            .attr('class', 'floor')
            .attr('i', function (d, i) {
                return i;
            })
            .attr('x', function (d, i) {
                return i % 30 * (rectWidth) + marginLeft;
            })
            .attr('y', function (d, i) {
                if (d.floor === 1)
                    return parseInt(i / 30) * (rectWidth) + marginTop;
                else    // 画二楼的底图，与一楼底图相隔20单位距离
                    return parseInt((i - 480) / 30) * (rectWidth) + marginTop + (rectWidth * 16 + 20);
            })
            .attr('width', rectWidth)
            .attr('height', rectWidth)
            .attr('stroke-width', strokeWidth)
            .attr('stroke', strokeColor)
            .attr('fill', function (d, i) {
                if (d.status === 0) return 'rgb(200, 200, 200)';
                else return 'white';
            });
    }

    /**
     * 显示两个楼层中各个场所的信息
     */
    function renderFloorDetail(floorDetail) {
        d3.select('svg')
            .selectAll('.floorDetail')
            .data(floorDetail)
            .enter()
            .append('rect')
            .attr('class', 'floorDetail')
            .attr('y', function (d) {
                if (d.floor === 1)
                    return d.x1 * (rectWidth) + marginTop;  // 留出顶边距
                else
                    return d.x1 * (rectWidth) + marginTop + (16 * rectWidth + 20);  // 留出顶边距
            })
            .attr('x', function (d) {
                return d.y1 * (rectWidth) + marginLeft; // 留出左边距
            })
            .attr('height', function (d) {
                return (d.x2 - d.x1) * rectWidth;       // 计算高度
            })
            .attr('width', function (d) {
                return (d.y2 - d.y1) * rectWidth;       // 计算宽度
            })
            .attr('stroke', 'black')        // 边框黑色
            .attr('stroke-width', 3)        // 边框线宽为3
            .style('cursor', 'pointer')
            .attr('fill', function (d, i) { // 根据区域类型填充颜色
                switch (d.type) {
                    case 'public area':
                        return "rgba(97, 149, 214, 0.7)";
                    case 'facility':
                        return "rgba(249, 167, 13, 0.7)";
                }
            })
            .on('mouseover', function () {
                d3.select(this)
                    .attr('fill', function (d) {
                        switch (d.type) {
                            case 'public area':
                                return "rgba(97, 149, 214, 0.9)";
                            case 'facility':
                                return "rgba(249, 167, 13, 0.9)";
                        }
                    })
            })
            .on('mouseout', function () {
                d3.select(this)
                    .attr('fill', function (d) {
                        switch (d.type) {
                            case 'public area':
                                return "rgba(97, 149, 214, 0.7)";
                            case 'facility':
                                return "rgba(249, 167, 13, 0.7)";
                        }
                    })
            })
            .on('click', function (d) {
                console.log(d);
                renderRangeSensorTable(d);
            })
        ;
        d3.select('svg')
            .selectAll('.detailText')
            .data(floorDetail)
            .enter()
            .append('text')
            .attr('class', 'detailText')
            .attr('y', function (d) {
                if (d.floor === 1)
                    return d.x1 * (rectWidth) + marginTop;  // 留出顶边距
                else
                    return d.x1 * (rectWidth) + marginTop + (16 * rectWidth + 20);  // 留出顶边距
            })
            .attr('x', function (d) {
                return d.y1 * (rectWidth) + marginLeft; // 留出左边距
            })
            .attr('dy', function (d) {
                return (d.x2 - d.x1) * rectWidth / 2 + 5;
            })
            .attr('dx', function (d) {
                return (d.y2 - d.y1) * rectWidth / 2;
            })
            .attr('text-anchor', 'middle')
            .text(function (d) {
                return d.name;
            })
    }

    /**
     * 显示一个半透明遮罩，辅助热力图的显示
     */
    function renderTransparentLayer() {
        let pos = [{'x': marginLeft, 'y': marginTop, 'floor': 1}, {
            'x': marginLeft,
            'y': (16 * rectWidth + 20) + marginTop,
            'floor': 2
        }];
        console.log('显示透明图层');
        d3.select('svg')
            .selectAll('.transparent')
            .data(pos)
            .enter()
            .append('rect')
            .attr('class', 'transparent')
            .attr('y', function (d) {
                if (d.floor === 1)
                    return d.y;
                else if (d.floor === 2)
                    return d.y;  // 留出顶边距
            })
            .attr('x', function (d) {
                return d.x; // 留出左边距
            })
            .attr('width', function (d) {
                return rectWidth * 30;
            })
            .attr('height', function (d) {
                return 16 * rectWidth;
            })
            .attr('fill', function (d) {
                return 'rgba(100,100,100,0.4)';
            })
    }

    /**
     * 渲染图例，接收参数为x、y坐标以及渐变两端的颜色
     */
    function renderHeatMapLegend(x, y, rgbA, rgbB) {
        let defs = d3.select('svg').append("defs");

        let linearGradient = defs.append("linearGradient")
            .attr("id", "linearColor")
            .attr("x1", "0%")
            .attr("y1", "0%")
            .attr("x2", "100%")
            .attr("y2", "0%");

        let stop1 = linearGradient.append("stop")
            .attr("offset", "0%")
            .style("stop-color", rgbA.toString());

        let stop2 = linearGradient.append("stop")
            .attr("offset", "100%")
            .style("stop-color", rgbB.toString());

        d3.select('svg')
            .append('rect')
            .attr('class', 'heatMapLegend')
            .attr('x', x)
            .attr('y', y)
            .attr('width', 200)
            .attr('height', 30)
            .style('fill', "url(#" + linearGradient.attr("id") + ")");

        let texts = ['1', '>=500'];     // 图例文字
        d3.select('svg')
            .selectAll('.legendText')
            .data(texts)
            .enter()
            .append('text')
            .attr('class', 'heatMapLegend')
            .attr('x', function (d, i) {
                if (i === 0) return x;
                return x + 180;
            })
            .attr('y', function (d, i) {
                return y + 45;
            })
            .text(function (d) {
                return d;
            })
    }

    /**
     * 获取每日参会id列表
     */
    function getPersonList(day) {
        db.query(
            'select `id` from `persons` where `day`= ? ',
            [day],
            (err, res, field) => {
                if (err) throw err;
                vm.idList = res;
                vm.personIdList = vm.idList.map(item => {//将idList映射成personIdList
                    return {value: item.id, label: item.id}; // 该数组中 存键值对{value: label: }
                });
                // console.log(res);
            }
        )
    }

    /**
     * 渲染热力图
     * @param heatData 表示热力图的数据
     * @param rgbA
     * @param rgbB
     */
    function renderHeatMap(heatData, rgbA, rgbB) {
        // console.log(heatData);
        console.log('渲染热力图...');
        let linear = d3.scaleLinear()
            .domain([0, 500])
            .range([0, 1]);
        let compute = d3.interpolate(rgbA, rgbB);

        removeHeatMap();

        d3.select('svg')
            .selectAll('.heat')
            .data(heatData)
            .enter()
            .append('rect')
            .attr('class', 'heat')
            .attr('y', function (d) {
                if (d.floor === 1)
                    return d.x * (rectWidth) + marginTop;
                else if (d.floor === 2)
                    return d.x * (rectWidth) + marginTop + (16 * rectWidth + 20);  // 留出顶边距
            })
            .attr('x', function (d) {
                return d.y * (rectWidth) + marginLeft; // 留出左边距
            })
            .attr('width', function (d) {
                return rectWidth;
            })
            .attr('height', function (d) {
                return rectWidth;
            })
            .attr('fill', function (d) {
                let rgb = compute(linear(d.cnt));
                return 'rgba' + rgb.substr(3, rgb.length - 4) + ',0.7)';    // 将比例尺计算出的rgb颜色转换成rgba颜色
            })
            .style('cursor', 'pointer')
            .on('click', function (d) {
                getIdListBySidAndDayAndTime(d.sid, d.day, d.time, d.cnt);
            })
    }

    /**
     * 根据sid, day和time查询此时此传感器范围中共有多少人，有哪些人
     * @param sid 传感器id
     * @param day 日期
     * @param time 时间（单位：秒）
     * @param cnt 范围内人数
     */
    function getIdListBySidAndDayAndTime(sid, day, time, cnt) {
        console.log(sid);

        db.query(
            'select `id`, `time` from `days_detail` where `sid`=? and `day`=? and `time` between ? and ? group by `id`',
            [sid, day, time, time + 600],
            (err, res, field) => {
                if (err) throw err;
                console.log(sid, '在', time, '时刻有人数', cnt);
                vm.sensorId = sid;
                vm.countPersons = cnt;
                vm.singleSensorTableVisible = true;
                for (let i = 0; i < res.length; i++) res[i].time = util.parseTime(res[i].time);
                vm.idList = res;
                console.log(res);
            }
        )
    }

    /**
     * 移除热力图
     */
    function removeHeatMap() {
        d3.select('svg')
            .selectAll('.heat')
            .remove();
    }

    /**
     * 销毁热力图
     * 销毁热力图操作不仅移除热力图，还移除半透明遮罩，从而使热力图完全从svg中移除
     */
    function destroyHeatMap() {
        removeHeatMap();
        d3.select('svg')
            .selectAll('.heatMapLegend')
            .remove();
        d3.select('svg')
            .selectAll('.transparent')
            .remove();

    }

    /**
     * 隐藏热力图
     */
    function hideHeatMap() {
        d3.select('svg')
            .selectAll('.heat')
            .attr('hidden', true);
        d3.select('svg')
            .selectAll('.transparent')
            .attr('hidden');
        d3.select('svg')
            .selectAll('.heatMapLegend')
            .attr('hidden');
    }

    /**
     * 显示热力图
     */
    function displayHeatMap() {
        d3.select('svg')
            .selectAll('.heat')
            .attr('hidden', null);
        d3.select('svg')
            .selectAll('.transparent')
            .attr('hidden', null);
        d3.select('svg')
            .selectAll('.heatMapLegend')
            .attr('hidden', null);
    }

    // 在一楼显示人员路径
    function renderRoutes(route, id = 1, idIdx) {//idIdx显示数组下标
        let threshold = d3.scaleThreshold()   //不同id显示不同颜色的路径  设置threshold阈值比例尺
            .domain([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
            .range(["60,179,113", "238,180,34", "238,180,180", "238,99,99", "72,118,255", "238,174,238", "159,182,205", "205,181,205", "162,181,205", "238,154,73"]);
        // let compute = d3.interpolate(rgbA, rgbB);
        console.log(idIdx);
        d3.select('svg')
            .selectAll('.route' + id)//class= route+id值 便于删除路径
            .data(route)
            .enter()
            .append('rect')
            .attr('class', 'route' + id)
            .attr('y', function (d) {
                if (d.floor === 1)
                    return d.x * (rectWidth) + marginTop;
                else if (d.floor === 2)
                    return d.x * (rectWidth) + marginTop + (16 * rectWidth + 20);  // 留出顶边距
            })
            .attr('x', function (d) {
                return d.y * (rectWidth) + marginLeft; // 留出左边距
            })
            .transition()                   // 在确定顶点坐标后设置动画
            .delay(function (d, i) {
                // return i * 100;  // 延迟100毫秒
                return d.duration * 1000 / 60 / 2;    // 按比例延迟显示
                // todo 添加手动设置播放速度的功能
            })
            .duration(function (d, i) {
                return 100;  // 时长100毫秒，一个紧接着一个冒出来
            })
            .attr('width', function (d) {   // 仅显示1楼路径
                return rectWidth;
            })
            .attr('height', function (d) {  // 仅显示1楼路径
                return rectWidth;
            })
            .attr('fill', function (d, i) { // 颜色由浅至深
                // return 'rgba(' + (23 - parseInt(i / route.length * (23))) + ', '
                //     + (195 - parseInt(i / route.length * (195))) + ', '
                //     + (41 - parseInt(i / route.length * (41))) + ', 0.5)'
                return 'rgba(' + threshold(idIdx) + ',0.7)';
            })
    }

    /**
     * 获取数据并显示范围人数统计图
     * @param d 选择的区域数据
     */
    function renderRangeSensorTable(d) {
        if (vm.dayValue == null) {
            vm.$message('请先选择日期');  // 未选择日期时弹出提示
            return;
        }
        vm.rangeSensorFigureVisible = true;
        let render = function () {                                                          // 用于渲染图像
            rangeSensorFigureOption.title.text = d.name;                                    // 设置图像标题
            rangeSensorFigureOption.series[0].data = [];                                    // 初始化图像数据
            for (let i = 0; i <= 90; i++) rangeSensorFigureOption.series[0].data.push(0);   // 初始化为0
            rangeSensorFigure.setOption(rangeSensorFigureOption);                           // 刷新图像
            console.log('计算', d.name, '统计数据');
            db.query(
                'select `time` from detail_day'+vm.dayValue+' where floor=? and `x` between ? and ? and `y` between ? and ? group by `id` order by `time`',
                [d.floor, d.x1, d.x2, d.y1, d.y2],
                (err, timeArr, field) => {
                    if (err) throw err;
                    console.log('数据已获得');
                    rangeSensorFigureOption.series[0].data = util.getTimePointArray(timeArr);   // 将原始数据转化为时间点数组
                    rangeSensorFigure.setOption(rangeSensorFigureOption);                       // 刷新图像
                }
            )
        };
        if (document.getElementById('rangeSensorFigure') == null)   // 第一次显示对话框时等待100毫秒再实例化图像，避免在
            setTimeout(function () {                                // 显示图像的div实例化前就进行图像的实例化
                rangeSensorFigure = echarts.init(document.getElementById('rangeSensorFigure')); // 实例化图像
                rangeSensorFigureOption.xAxis.data = util.timeDataGen();                        // 初始化图像坐标轴
                render();   // 画图
            }, 100);
        else
            render();       // 在#rangeSensorFigure存在时可以直接画图
    }

    let floor = StaticData.floor;
    let floorDetail = StaticData.floorDetail;
    let rangeSensorFigureOption = {
        title: {
            text: '范围统计',
            x: 'center'
        },
        xAxis: {
            type: 'category',
            data: [],
        },
        yAxis: {
            name: '人数（人）',
            type: 'value',
        },
        series: [{
            data: [],
            type: 'line'
        }],
        dataZoom: [
            {
                type: 'slider',
                show: true,
                xAxisIndex: [0],
                start: 1,
                end: 100
            },
            {
                type: 'inside',
                xAxisIndex: [0],
                start: 1,
                end: 100
            },
        ]
    };
    let vm = null;
    let rangeSensorFigure = null;

    const rectWidth = 25;   // 方块的边长
    const strokeWidth = 1;  // 边框的宽度
    const strokeColor = "rgb(149, 149, 149)";
    const marginLeft = 10;
    const marginTop = 5;
    const timeInterval = 600;   // 600sec，即十分钟时间间隔
    const baseTime = 6 * 3600;
    const linearGradientStartRgb = d3.rgb(166, 192, 254);   // 渐变起点颜色
    const linearGradientEndRgb = d3.rgb(246, 211, 101);     // 渐变终点颜色


    //===============  for testing  =================
    const idForTest = [18473];
    //===============  for testing  =================

    let svgWidth = 1010, svgHeight = 900;

    export default {
        name: "SvgMap",
        data() {
            return {
                dayValue: null,
                timePointValue: null,
                loading: false,
                personValue: [],
                routeValue: null,
                disableTimeSelect: true,
                disableIdSelect: false,
                options1: [
                    {
                        value: '1',
                        label: 'day1'
                    }, {
                        value: '2',
                        label: 'day2'
                    }, {
                        value: '3',
                        label: 'day3'
                    }
                ],
                options2: StaticData.options2,
                personIdOptions: [],
                personIdValues: [],
                personIdList: [],
                singleSensorTableVisible: false,
                rangeSensorFigureVisible: false,
                rangeName: "",
                sensorId: null,
                countPersons: 0,
                idList: [],
                heatMapSwitch: false,
            }
        },
        watch: {
            timePointValue: function (newTimePoint, oldTimePoint) {
                console.log("dayval: " + this.dayValue, "timpointval: " + this.timePointValue);
                if (this.heatMapSwitch && this.dayValue != null)
                    this.showHeatMap(this.dayValue, this.timePointValue);
            },
            dayValue: function (newDay, oldDay) {
                getPersonList(this.dayValue);//获取该日id列表
                if (this.heatMapSwitch)
                    this.showHeatMap(this.dayValue, this.timePointValue);
            },
            heatMapSwitch: function (newVal, oldVal) {
                console.log('热力图开关：', newVal);
                if (this.heatMapSwitch) {             //开启热力图开关
                    this.disableIdSelect = true;  //关闭人员筛选框
                    this.disableTimeSelect = false;   //开启日期筛选框
                    this.initHeatMap();             //初始渲染 热力图
                    this.removeRoute(this.personValue); //把地图上的人员路径删去
                } else {
                    this.disableTimeSelect = true;
                    this.disableIdSelect = false;
                    destroyHeatMap();//删去热力图
                    this.findRouteById(this.personValue, this.dayValue);//重新渲染id筛选框中id的路径图
                }
            },
            personValue: function (newRouteValue, oldRouteValue) {
                if (!this.heatMapSwitch) {  //开启路径图
                    let filter1 = newRouteValue.filter(item => {  //filter1: 在每一次添加id时 选出新增的id值
                        return oldRouteValue.indexOf(item) === -1;
                    });
                    // console.log(filter1);
                    this.findRouteById(filter1, this.dayValue, this.personValue.indexOf(filter1[0]));//渲染该id的路径 并传入id在personValue数组中的index：确定路径颜色
                    let filter2 = oldRouteValue.filter(item => {//filter2: 在每一次删除id时 得到删去的id值
                        return newRouteValue.indexOf(item) === -1;
                    });
                    this.removeRoute(filter2); //删除对应id值的路径
                    console.log(filter2);
                }
            },
        },
        mounted() {
            vm = this;
            let svg = d3.select('svg')         // 设置svg元素
                .attr('width', svgWidth)       // 设置宽度
                .attr('height', svgHeight);    // 设置高度

            renderFloorMap(floor);             // 渲染地图底图
            renderFloorDetail(floorDetail);    // 显示各个区域
            // renderFloorDetail(floorDetail);    // 显示各个区域
            // this.showHeatMap(1, 32);
        },
        methods: {
            /*
            * 删除用户的路径
             */
            removeRoute(ids) {
                for (let i = 0; i < ids.length; i++)
                    d3.selectAll('.route' + ids[i])
                        .remove();
            },
            /*
            在id筛选框中 输入数字 得到所有id值
             */
            remoteMethod(query) {
                if (query !== '') {
                    this.loading = true;
                    setTimeout(() => {
                        this.loading = false;
                        // console.log(this.personIdList);
                        this.personIdOptions = this.personIdList.filter(item => {  //将包含query的 从数据库中返回的personIdList 数组赋給personIDOptions
                            return item.label.toString().indexOf(query) > -1;
                        });
                        // console.log(this.personIdOptions);
                    }, 200);
                } else {
                    this.personIdOptions = []; //若query为0 则将 personIDOptions数组赋空
                }
            },
            /**
             * 初始化热力图
             */
            initHeatMap: function () {
                renderTransparentLayer();   // 显示透明遮罩
                renderHeatMapLegend(marginLeft + 30 * rectWidth + 20, marginTop, linearGradientStartRgb, linearGradientEndRgb); // 显示图例
                this.showHeatMap();         // 显示热力图
            },

            /**
             * 用于查询数据并调用渲染热力图函数
             */
            showHeatMap: function () {
                let day = this.dayValue, timePoint = this.timePointValue;
                let startTime = baseTime + timePoint * timeInterval;
                let endTime = startTime + timeInterval;
                console.log(startTime, endTime);
                this.$db.query(
                    'select sid, day, x, y, floor, time, count(*) cnt from `days_detail` where `day`=? and `time` between ? and ? group by `sid`',
                    [day, startTime, endTime],    // 取第timePoint个时间间隔内的人数, 此处两个表达式的值单位都为秒
                    (err, res, field) => {
                        if (err) throw err;
                        for (let i = 0; i < res.length; i++)
                            res[i].time = baseTime + timePoint * timeInterval;
                        if (this.heatMapSwitch)
                            renderHeatMap(res, linearGradientStartRgb, linearGradientEndRgb);
                    }
                )
            },

            findRouteById: function (ids, day, colorNum) {    // 传入人员id和日期 以及 id在数组中的位置，显示人员行走路径
                for (let idIdx = 0; idIdx < ids.length; idIdx++) {
                    this.$db.query(
                        'select * from days a join sensors using(`sid`) where `id`=? and `day`=? order by `time`',
                        [ids[idIdx], day],
                        (err, res, field) => {
                            if (err) throw err;
                            for (let resIdx = 0; resIdx < res.length; resIdx++) {       // 遍历查出来的每条记录
                                res[resIdx].duration = 0;                               // 将记录中有用的部分取出，
                            }
                            for (let resIdx = res.length - 1; resIdx >= 0; resIdx--) {  // 计算每个路径点的停留时长
                                res[resIdx].duration = res[resIdx].time - res[0].time;
                            }
                            if (colorNum === undefined)
                                renderRoutes(res, ids[idIdx], idIdx); // 获得路径后显示出来
                            else
                                renderRoutes(res, ids[idIdx], colorNum);
                            console.log(idIdx);
                        }
                    )
                }
            }
        }
    }
</script>

<style scoped>
    svg {
        margin: 20px 0;
    }
    .svgContainer{
        display: flex;
        align-items: center;
        justify-content: center;
    }
</style>
