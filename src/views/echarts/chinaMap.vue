<template>
  <div class="">
    <div class="banner">
      <div class="banner-top">
        <p class="banner-part">设备接入总数：<span v-text="this.equipments"></span></p>
        <p class="banner-part">设备上云企业数：<span v-text="this.companies"></span></p>
      </div> 
      <div class="border-box">
        <div class="border-part">
          <p class="part-icon running"></p>
          <p class="part-percent"><span class="rate" v-text="this.runningRate"></span><span class="mini">%</span></p>
          <p class="part-name">运行率</p>
        </div>
        <div class="border-part">
          <p class="part-icon break"></p>
          <p class="part-percent"><span class="rate" v-text="this.errRate"></span><span class="mini">%</span></p>
          <p class="part-name">故障率</p>
        </div>
        <div class="border-part">
          <p class="part-icon wait"></p>
          <p class="part-percent"><span class="rate" v-text="this.waitRate"></span><span class="mini">%</span></p>
          <p class="part-name">待机率</p>
        </div>
      </div>
    </div>
    <!-- <div class="title">Echarts中国地图三级钻取</div> -->
    <div class="box">
      <button class="backBtn">
        <span @click="backChina()">返回中国</span>
      </button>
      <div id="mapChart" class="chart"></div>
    </div>
  </div>
</template>

<script>
import cityMap from "@/js/china-main-city-map.js";
import echarts from "echarts";
import axios from "axios";
import Vue from "vue";

// axios.defaults.baseURL = process.env.API_ROOT

var chinaData = [];
var cityData = [];

var geoCoordMap = {};

//中国地图（第一级地图）的ID、Name、Json数据
var chinaId = 100000;
var chinaName = "china";
var chinaJson = null;

//Echarts地图全局变量，主要是在返回上级地图的方法中会用到
var myChart = null;

export default {
  name: "chinaMap",
  data() {
    return {
      equipments: 0,
      companies: 0,
      runningRate: 0,
      errRate: 0,
      waitRate: 0,
      mapData: [],
      cityData: []
    }
  },
  props: {
    msg: String
  },
  mounted: function() {
    this.mapChart("mapChart");
    this.getEquipments();
    this.getCompanies();
    this.getMapData();
  },
  methods: {
    /**
     * 设备接入总数接口,运行率、故障率、待机率
     */
    getEquipments () {
      axios
        .get("http://172.16.18.215:8080/api/1.0/iot/getAll", {}).then(res => {
          // console.log(res)
          let data = res.data.results
          this.equipments = data.total.toLocaleString();
          // this.runningRate = data.yxl.replace(/[%]/g,'');
          // this.errRate = data.gzl.replace(/[%]/g,'');
          // this.waitRate = data.djl.replace(/[%]/g,'');
          this.runningRate = data.yxl;
          this.errRate = data.gzl;
          this.waitRate = data.djl;
        })
    },
    /**
     * 设备上云企业数
     */
    getCompanies () {
      axios
        .get("http://172.16.18.215:8080/api/1.0/iot/getJoinOrgs", {}).then(res => {
          let data = res.data.results;
          this.companies = data.total.toLocaleString();
        })
    },
    /**
     * 获取全国地图数据
     */
    getMapData () {
      axios
        .get("http://172.16.18.215:8080/api/1.0/iot/getJoinProvincesAndDevsMap", {}).then(res => {
          // console.log(res.data.results)
          let response = res.data.results;
          let countryData = dealMapData(response.listProvinces, response.listProvincesEqu);
          chinaData = countryData;
          this.mapChart("mapChart");
          // console.log("全国数据：",countryData)
        })
    },
    /**
     * 返回中国
     */
    backChina () {
      var list = document.getElementById("mapChart")
      list.childNodes[1].style.display = 'none';
      registerAndsetOption(myChart, chinaId, chinaName, chinaJson, chinaData);
    },
    /**
     * Echarts地图
     */
    mapChart(divid) {
      axios.get("./static/json/map/" + chinaId + ".json", {}).then(response => {
        const mapJson = response.data;
        chinaJson = mapJson;
        myChart = echarts.init(document.getElementById(divid));
        registerAndsetOption(myChart, chinaId, chinaName, mapJson, chinaData);
        myChart.on("click", function(params) {
          showTip (true)
          if (Boolean(params.value)){
            showTip (true)
          } else {
            showTip (false)
          }        
        })      
      });
    }
  }
};
/**
 * Echarts地图setOption
 */
function setOption(myChart, name, mapJson, data) {
  myChart.setOption({
    title: {
      text: name === 'china' ? '中国' : name,
      left: 'center',
      textStyle: {
        color: '#666',
        top: 'auto',
        fontSize: 12
      }
    },
    tooltip: {
      // show: true,
      triggerOn: 'click',
      // trigger: 'item',
      padding: 0,
      enterable: true,
      transitionDuration: 1,
      textStyle: {
        color: '#666',
        decoration: 'none',
      },
      formatter: function(params) { 
        if (Boolean(params.value)){
          var _html = '';
          // _tipHtml 提示框数据
          var _tipHtml = '<p style="margin:0;color:#fdd01e">'+params.data.name+'</p>';
          
          if (params.data.tipData === "undefined"){
            return
          }
          params.data.tipData.forEach((val, k) => {
            _tipHtml += '<p style="margin:0;">'+val.name+":"+val.value.toLocaleString()+'</p>'
          })
          _html = '<div id="next" style="background:#fff;border-radious:4px;padding:4px 10px;font-size:12px;">'+_tipHtml+'</div>'
            setTimeout(function() {
              document.getElementById("next").addEventListener("click", function() {
                var list = document.getElementById("mapChart")
                list.childNodes[1].style.display = 'none';
                var cityId = cityMap[params.name];
                
                // if (cityId) {
                  if (name === "china"){
                    
                    if (params.name === "北京" || params.name === "上海" || params.name === "天津" || params.name === "重庆") {
                      if (params.name === "北京") {
                        cityId = "110100"
                      } else if (params.name === "上海") {
                        cityId = "310100"
                      } else if (params.name === "天津") {
                        cityId = "120100"
                      } else if (params.name === "重庆") {
                        cityId = "500100"
                      }
                      let obj = {"type" : "DeviceMap","province": name,"cityName": params.name, "cityId": cityId};
                      // console.log(JSON.stringify(obj));
                      getIosAnd(JSON.stringify(obj));
                    } else {
             
                      //代表有下级地图
                      axios
                        .get("./static/json/map/" + cityId + ".json", {})
                        .then(response => {      
                          showTip(false);                                     
                          const mapJson = response.data;
                          registerAndsetOption(
                            myChart,
                            cityId,
                            params.name,
                            mapJson,
                            chinaData
                          );

                          let provinceName = params.name                         
                          getCityData (provinceName,myChart,cityId,params.name,mapJson)
                        });
                            
                    }        
                    
                  } else {
                    let obj = {"type" : "DeviceMap","province": name,"cityName": params.name,"cityId": cityId};
                    // console.log(JSON.stringify(obj));
                    getIosAnd(JSON.stringify(obj));
                  }
              })
            })
          return _html;
        }
      }
    },
    visualMap: {
      type: 'piecewise',
      pieces: [
        {
          max: 0,
          label: '无数据',
          color: '#2c9a42'
        },
        {
          min: 1,
          label: '有数据',
          color: '#4987f6'
        }
      ],
      show: false,
      calculable: true,
      seriesIndex:'0'
    },
    series: [
      {
        type: "map",
        map: name,
        roam: false,
        zoom: 1.1,
        itemStyle: {
          normal: {
            areaColor: "#abd4fa",
            borderColor: "#fff",
            borderWidth: 1
          },
          emphasis: {
            label: {
              color: "#959fa7",
            },
            areaColor: "#fdd01e",
            borderColor: "#1dc199",
            borderWidth: 1
          }
        },
        data: data
      }
    ]
  })
}
/**
 *
 * @param {*} myChart
 * @param {*} id        省市县Id
 * @param {*} name      省市县名称
 * @param {*} mapJson   地图Json数据
 */
function registerAndsetOption(myChart, id, name, mapJson, data) {
  showTip (false);
  geoCoordMap = {}
  mapJson.features.forEach((v, i) => {
    // 地区名称
    var name = v.properties.name;
    // 地区经纬度
    geoCoordMap[name] = v.properties.cp;
  });
  
  echarts.registerMap(name, mapJson);
  setOption(myChart, name, mapJson, data)
}

//调用移动安卓方法
function getIosAnd(opts){
  var u = navigator.userAgent;
  var isAndroid = u.indexOf('Android') > -1 || u.indexOf('Adr') > -1; //android终端
  var isiOS = !!u.match(/\(i[^;]+;( U;)? CPU.+Mac OS X/); //ios终端
  if(isAndroid){
    window.htyw.actionFromJsWithParam(opts);
  }else if(isiOS){
    window.webkit.messageHandlers.AppModel.postMessage(opts);
  }
};

//是否显示地图的tooltip
function showTip (isShow) {
  myChart.setOption({tooltip: {show: isShow}});
}

/**
 * 地图接口数据处理
 */
function dealMapData (equ, com) {
  let _data1 = [], _data2 = [];
  let companiesCounts = com // 企业总数
  let equipmentsCounts = equ // 设备总数
  companiesCounts.filter((val, i) => {
    let obj = new Object(null);
    if (val.name) {
      obj.name = val.name;
      obj.tipData = [{name:"企业总数", value: val.value}];
      _data1.push(obj)
    }           
  })
  equipmentsCounts.filter((val, i) => {
    let obj = new Object(null);
    if (val.name) {
      obj.name = val.name;
      obj.tipData = [{name:"设备总数", value: val.value}];
      _data2.push(obj)
    }  
  })

  let mapData = [];
  mapData = _data1.concat(_data2);
  mapData = mapData.reduce((acc, item, index, source) => {
    if (!acc[item.name]) {
      acc[item.name] = item
      acc[item.name].value = 10
    } else {
      acc[item.name].tipData.push(item.tipData[0]) 
      acc[item.name].value = 10
    }
    return acc;
  }, {})
  
  let _mapData = [];
  mapData = Object.values(mapData).forEach(item => {
    _mapData.push(item);
  })

  return _mapData;
}

/**
 * 地图市级数据接口请求
 */
function getCityData (provinceName,myChart,cityId,name,mapJson) {
  // console.log(provinceName)
  if (provinceName.indexOf("%") != -1) {
    provinceName = decodeURIComponent(provinceName);
  } else {
    provinceName = provinceName
  }

  axios.get(`http://172.16.18.215:8080/api/1.0/iot/getJoinCitysAndDevs?province=${provinceName}`, {}).then(res => {
    let item = res.data.results   
    cityData = dealMapData(item.listCity, item.listCitysEqu);
    console.log(provinceName,cityData);     
    registerAndsetOption(
      myChart,
      cityId,
      name,
      mapJson,
      cityData
    );             
  }) 
}

</script>

<style scoped>
p,div {
  margin: 0;
  padding: 0;
}
.banner {
  position: absolute;
  width: 100%;
  height: 20vh;
  top: 0;
  left: 0;
  background: url(../../assets/banner.png) no-repeat;
  background-size: 100% 100%;
}
.banner-top {
  display: flex;
}
.banner-part {
  font-weight: bold;
  margin-top: 8%;
  height: 20px;
  flex: 1;
  color: #fff;
  text-align: center;
  font-size: 0.9em;
}
.border-box {
  display: flex;
  height: 95%;
  width: 100%;
  position: absolute;
  bottom: -50%;
  left: 0;
  background: url(../../assets/border.png) no-repeat;
  background-size: 100% 100%;
}
.border-part {
  flex: 1;
  text-align: center;
  color: #467bc0;
}
.part-icon {
  margin-top: 8%;
  height: 30%;
}
.running {
  background: url(../../assets/running.png) no-repeat center;
  background-size: 20% auto;
}
.break {
  background: url(../../assets/break.png) no-repeat center;
  background-size: 20% auto;
}
.wait {
  background: url(../../assets/wait.png) no-repeat center;
  background-size: 20% auto;
}
.part-percent {
  height: 30%;
  box-sizing: border-box;
  padding-top: 5%;
}
.part-percent .rate {
  font-size: 1.2em;
  font-weight: bold;
}
.part-percent span.mini {
  font-size: 0.5em;
}
.part-name {
  height: 30%;
  font-size: 0.75em;
}

.title {
  width: 100%;
  height: 10vh;
  text-align: center;
  color: #fff;
  font-size: 2em;
  line-height: 10vh;
}
.box {
  position: absolute;
  width: 100%;
  height: 60vh;
  top: 32%;
  left: 0;
}
.chart {
  position: relative;
  height: 100%;
  top: 10%;
}
.backBtn {
  position: absolute;
  top: 0;
  background-color: #467bc0;
  border: 0;
  color: #fff;
  height: 27px;
  font-family: Microsoft Yahei;
  font-size: 0.8em;
  cursor: pointer;
}
.myBlog {
  position: absolute;
  top: 2px;
  right: 5%;
  display: block;
  border: 2px solid #262a53;
}
.myBlog a {
  text-decoration: none;
  display: inline-block;
  color: #fff;
  padding: 5px;
  font-size: 1em;
}

.myBlog a img {
  vertical-align: middle;
  height: 20px;
  width: 20px;
  border-radius: 50%;
  margin: -5px 5px auto auto;
}
.bottom {
  position: absolute;
  width: 100%;
  height: 5%;
  line-height: 100%;
  left: 0;
  bottom: 0%;
  text-align: center;
}
</style>
