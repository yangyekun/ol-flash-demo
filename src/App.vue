<template>
  <div class="toggle">
    <span>显示控制: </span>
    <input type="radio" name="showType" id="none" :value="0" v-model="shtp">
    <label for="none">无</label>
    <input type="radio" name="showType" id="showDzx" :value="1" v-model="shtp">
    <label for="showDzx">等值线</label>
    <input type="radio" name="showType" id="showDzm" :value="2" v-model="shtp">
    <label for="showDzm">等值面</label>
  </div>
  <div class="map-container" id="wMap"></div>
  <span class="curZoom" style="position: absolute; bottom: 8px; left: 140px;">地图层级: {{currentZoom}}</span>
</template>

<script setup>
import {ref, onMounted, watch} from "vue";
import { Map, View } from "ol";
import { Tile as TileLayer, Vector as VectorLayer} from "ol/layer";
import { Vector as VectorSource} from "ol/source";
import {ScaleLine} from 'ol/control';
import Feature from 'ol/Feature';
import { Point, LineString, Polygon } from 'ol/geom';
import { getVectorContext } from "ol/render";
import {Fill, Style, Stroke, Circle, Text} from 'ol/style';
import "ol/ol.css";
import EsriJSON from 'ol/format/EsriJSON';
import WMTS, {optionsFromCapabilities} from 'ol/source/WMTS';
import WMTSCapabilities from 'ol/format/WMTSCapabilities';

const anhuiUrl = 'http://60.174.203.118:6080/arcgis/rest/services/AnHuiShuiXinXi/AnHuiShuiXinXi_SXDT/MapServer/WMTS/1.0.0/WMTSCapabilities.xml';

let map, mapView;

let isClick = ref(false)
let currentZoom = ref(7);
let loadCount = ref(0);

// 显示控制
let shtp = ref(1)

onMounted(() => {
  initMap();
})

const initMap = () => {
  mapView = new View({
    enableRotation: false,
    projection: "EPSG:4326",
    center: [117.021597, 31.552257],
    minZoom: 6,
    zoom: 7
  })
  map = new Map({
    target: 'wMap',
    controls: [],
    layers: [],
    view: mapView
  })
  map.addControl(new ScaleLine());
  // 添加安徽底图
  fetch(anhuiUrl)
  .then(function (response) {
    return response.text();
  })
  .then(function (text) {
    const result = new WMTSCapabilities().read(text);
    const layerName = result.ServiceIdentification.Title;
    let options = optionsFromCapabilities(result, {
      crossOrigin: "anonymous",
      layer: layerName
    });
    options.tilePixelRatio = 2;
    map.addLayer(
      new TileLayer({
        source: new WMTS(options),
      })
    );
  });

  // 监听鼠标滚轮
  mapView.on("change:resolution", e => {
    currentZoom.value = mapView.getZoom().toFixed(1);
  })

  // 地图点击事件，cleTm防止多次执行相同操作
  let cleTm;
  map.on("singleclick", e => {
    // 当前点击的经纬度
    console.log(e.coordinate);
    map.forEachFeatureAtPixel(e.pixel, (feature, layer) => {
      if (isClick.value) {
        // 防止多次触发
        return;
      }
      isClick.value = true;
      if (cleTm) {
        clearTimeout(cleTm)
      }
      // 200ms后把isClick重新改为false，不然点第二个点会无反应，被上面return掉
      cleTm = setTimeout(() => {
        isClick.value = false;
      }, 200);
      // 获取当前点击的feature，以便进行后续操作
      console.log(feature);
    },{ hitTolerance: 20 });
  })

  // 鼠标划过点的时候变成手形
  map.on('pointermove',(e)=>{
    let feature = map.forEachFeatureAtPixel(e.pixel,(feature)=>{
      return feature
    })
    // 线、面要素不做鼠标移入样式修改
    if(feature == undefined || feature.getGeometry().getType() !='Point'){
      map.getTargetElement().style.cursor = 'auto'
    }else{
      map.getTargetElement().style.cursor = 'pointer'
    }
  })
  // 绘制闪烁点图层，r1 r2内圈外圈，style对应内圈外圈的样式，
  // 最中间圆点保持不动，让r1和r2向外扩散
  let rainFeature = [];
  let rains = [{lgtd: 117.946241, lttd: 32.458797},{lgtd: 118.499925, lttd: 32.395283}]
  let rainStyle = new Style({
    image: new Circle({
      fill: new Fill({
        color: "#0000ff",
      }),
      stroke: new Stroke({
        color: '#fff',
        width: 1,
      }),
      radius: 4,
    })
  })
  let r1 = 4;
  let r2 = 5;
  let haloStyle1 = new Style({
    image: new Circle({
      fill: new Fill({
        color: "transparent",
      }),
      stroke: new Stroke({
        color: '#0000ff',
        width: 0.5,
      }),
      radius: r1,
    })
  })
  let haloStyle2 = new Style({
    image: new Circle({
      fill: new Fill({
        color: "transparent",
      }),
      stroke: new Stroke({
        color: '#0000ff',
        width: 0.5,
      }),
      radius: r2,
    })
  })
  for (let i = 0; i < rains.length; i++) {
    let feature = new Feature({
      geometry: new Point([Number(rains[i].lgtd), Number(rains[i].lttd)]),
      name: 'rainIcon'
    });
    let haloFeature1 = new Feature({
      geometry: new Point([Number(rains[i].lgtd), Number(rains[i].lttd)]),
      name: 'haloFeature1'
    });
    let haloFeature2 = new Feature({
      geometry: new Point([Number(rains[i].lgtd), Number(rains[i].lttd)]),
      name: 'haloFeature2'
    });
    feature.setStyle(rainStyle)
    haloFeature1.setStyle(haloStyle1)
    haloFeature2.setStyle(haloStyle2)
    rainFeature.push(feature)
    rainFeature.push(haloFeature1)
    rainFeature.push(haloFeature2)
  }
  const rainSource = new VectorSource({
    features: rainFeature
  })
  const rainLayer = new VectorLayer({
    source: rainSource,
    type: "rain",
    zIndex: 20
  })
  map.addLayer(rainLayer);

  // 让点闪烁起来，上一步仅绘制了图层，也就是第一个画面(帧)，还并不会闪
  rainLayer.on("postrender", evt => {
    if (r1 >= 5.5) {
      r1 = 4
    }
    if (r2 >= 6.5) {
      r2 = 5
    }
    let haloStyle1 = new Style({
      image: new Circle({
        fill: new Fill({
          color: "transparent",
        }),
        stroke: new Stroke({
          color: '#0000ff',
          width: 0.5,
        }),
        radius: r1,
      })
    })
    let haloStyle2 = new Style({
      image: new Circle({
        fill: new Fill({
          color: "transparent",
        }),
        stroke: new Stroke({
          color: '#0000ff',
          width: 0.5,
        }),
        radius: r2,
      })
    })
    const fets = rainLayer.getSource().getFeatures();
    const haloFets = fets.filter(item => item.get("name") != "rainIcon");
    haloFets.forEach(feat => {
      if (feat.get("name") === "haloFeature1") {
        feat.setStyle(haloStyle1)
      } else if (feat.get("name") === "haloFeature2") {
        feat.setStyle(haloStyle2)
      }
    })
    r1+=0.01;
    r2+=0.01;
    // 调用layer的changed方法，让他不断绘制下一个画面，即形成闪烁
    rainLayer.changed();
  })

  // 水体闪烁，适用于河流，水库，湖泊等闪烁
  // 通常warns为空，经过特定条件判断后，会得到一些报警点，这里举例就1个报警点
  const warns = ["62915400"]
  if (warns.length) {
    let warnFeatures = [];
    // 当前线宽和背景色
    let curLineWidth = 1;
    let colorR = 255;
    let colorG = 255;
    let colorB = 0;
    // flag用于标志是放大还是缩回
    let flag = 1;
    //同样先定义图层和数据源
    const iconSource = new VectorSource({
      features: warnFeatures
    })
    const iconLayer = new VectorLayer({
      source: iconSource,
      type: "rain",
      zIndex: 20
    })
    // 这里的水体文件是arcgis生成的json文件，文件放在public/xxxxxxx.json
    // 先使用fetch请求json文件，然后定义水体样式，填充色和描边色
    warns.forEach(item => {
      const jsonurl = "./" + item + ".json"
      fetch(jsonurl).then(res => {
        return res.json()
      }).then(res => {
        const fill = new Fill({
          color: `rgb(${colorR}, ${colorG}, ${colorB})`,
        });
        const stroke = new Stroke({
          color: '#ff2e2e',
          width: curLineWidth,
        });
        let iconStyle = new Style({
          fill: fill,
          stroke: stroke,
        })
        // 这里样式定义完毕，这种json文件是arcgis生成的，需要用EsriJSON类来加载feature
        const esri = new EsriJSON();
        let feature = esri.readFeature(res.features[0]);
        // 应用样式
        feature.setStyle(iconStyle)
        // 把feature添加到数据源里
        iconSource.addFeature(feature)
        // 结束后count加1，这里只有一个报警点，有多个count就一直加
        loadCount.value++;
      })
    })
    // 将图层加到地图上，注意此时只是把所有报警点加到图层上了，并没有闪烁
    map.addLayer(iconLayer);
    // 水体闪烁方法，同样使用postrender事件，将线宽和背景色逐渐变化
    const requestFlash = () => {
      iconLayer.on("postrender", evt => {
        curLineWidth += flag * 0.01;
        colorR += flag * 0;
        colorG += flag * -0.5;
        colorB += flag * 0.3;
        const fill = new Fill({
          color: `rgb(${colorR}, ${colorG}, ${colorB})`,
        });
        const stroke = new Stroke({
          color: '#ff2e2e',
          width: curLineWidth,
        });
        let iconStyle = new Style({
          fill: fill,
          stroke: stroke,
        })
        let vectorContext = getVectorContext(evt);
        vectorContext.setStyle(iconStyle);
        const fets = iconLayer.getSource().getFeatures();
        fets.forEach(feat => {
          vectorContext.drawGeometry(feat.getGeometry());
        })
        // 当线宽缩小到1，flag改为1，此时开始放大
        if (curLineWidth.toFixed(0) == "1") {
          flag = 1
        }
        // 当线宽放大到3，flag改为-1，此时开始缩小
        if (curLineWidth.toFixed(0) == "3") {
          flag = -1;
        }
        // 调用layer的changed方法，进行绘制
        iconLayer.changed();
      })
    }
    // 监听已加载的json文件，已加载个数 等于 报警点个数的时候才开始闪烁水体
    watch(loadCount, val => {
      if (val === warns.length) {
        requestFlash();
      }
    })
  }


  // 等值线等值面
  let dzxLayer, dzmLayer;
  fetch("./test.xml").then(res => {
    return res.text();
  }).then(res => {
    // 等值线数据以 9999,10  这样的格式为某一段的结束标记
    // 115.6801,31.20302|115.6648,31.2345|115.6549,31.24864|.....
    const dzxStr = res.slice(res.indexOf("<dzx>") + 7, res.indexOf("</dzx>") - 1)
    const dzxArr = dzxStr.split("|")
    let dataArr = [];
    let obj = {
      val: 0,
      data: []
    }
    dzxArr.forEach(item => {
      const tmpArr = item.split(",");
      if (Number(tmpArr[0]) !== 9999) {
        obj.data.push([Number(tmpArr[0]), Number(tmpArr[1])])
      } else {
        obj.val = Number(tmpArr[1])
        dataArr.push(obj)
        obj = {
          val: 0,
          data: []
        }
      }
    })
    let dzxFeature = [];
    dataArr.forEach(item => {
      if (item.data.length) {
        const startText = new Feature({
          geometry: new Point([item.data[0][0], item.data[0][1]]),
          name: "dzxStart"
        })
        const endText = new Feature({
          geometry: new Point([item.data[item.data.length - 1][0], item.data[item.data.length - 1][1]]),
          name: "dzxEnd"
        })
        const lineFet = new Feature({
          geometry: new LineString(item.data),
          name: "dzxLine"
        })
        let style = new Style({
          text: new Text({
            fill: new Fill({
              color: "#000ff8",
            }),
            font: "12px 黑体",
            text: item.val + ""
          })
        })
        startText.setStyle(style)
        endText.setStyle(style)
        style = new Style({
          stroke: new Stroke({
            color: '#ff071b',
            width: 2,
          })
        })
        lineFet.setStyle(style)
        dzxFeature.push(startText)
        dzxFeature.push(endText)
        dzxFeature.push(lineFet)
      }
    })
    const dzxSource = new VectorSource({
      features: dzxFeature
    })
    dzxLayer = new VectorLayer({
      source: dzxSource,
      type: "dzx",
      zIndex: 19
    })
    map.addLayer(dzxLayer);


    // 等值面数据以 a,10,1|117....  这样的格式为某一个面的结束标记
    // 117.7487,33.83841:117.7446,33.83484:117.7404,33.83126:117.7362,33.82769:117.7362.....
    const color = {
      "-10": "#CCFF66",
      "10": "#00ffff",
      "-25": "#00ffff",
      "25": "#00CCFF",
      "-50": "#00CCFF",
      "50": "#0099FF",
      "-100": "#0099FF",
      "100": "#0066FF",
    }
    const dzmStr = res.slice(res.indexOf("<dzm>") + 7, res.indexOf("</dzm>") - 1)
    const dzmArr = dzmStr.split(":")
    let dzmDatas = [];
    let dzmObj = {
      val: 0, // 等值面绘制中，未使用到val
      color: "", // 面的颜色
      data: []
    }
    dzmArr.forEach(item => {
      const tmpArr = item.split(",");
      if (tmpArr[0] !== "a") {
        dzmObj.data.push([Number(tmpArr[0]), Number(tmpArr[1])])
      } else {
        dzmObj.val = Number(tmpArr[1])
        const key = Number(tmpArr[1]) * Number(tmpArr[2].split("|")[0]) + ""
        dzmObj.color = color[key]
        dzmDatas.push(dzmObj)
        dzmObj = {
          val: 0,
          color: "",
          data: []
        }
      }
    })
    let dzmFeature = [];
    dzmDatas.forEach(item => {
      if (item.data.length) {
        const polyFet = new Feature({
          geometry: new Polygon([item.data]),
          name: "dzmPoly"
        })
        let style = new Style({
          fill: new Fill({
            color: item.color,
          })
        })
        polyFet.setStyle(style)
        dzmFeature.push(polyFet)
      }
    })
    const dzmSource = new VectorSource({
      features: dzmFeature
    })
    dzmLayer = new VectorLayer({
      source: dzmSource,
      type: "dzm",
      visible: false,
      zIndex: 19
    })
    map.addLayer(dzmLayer);

    watch(() => shtp.value, val => {
      switch (val) {
        case 0:
          dzxLayer.setVisible(false);
          dzmLayer.setVisible(false);
          break;
        case 1:
          dzxLayer.setVisible(true);
          dzmLayer.setVisible(false);
          break;
        case 2:
          dzxLayer.setVisible(false);
          dzmLayer.setVisible(true);
          break;     
        default:
          break;
      }
    })

  })

}


</script>

<script>
import { defineComponent } from "vue";
export default defineComponent({
  name: "waterMap"
});
</script>

<style>
html,body {
  height: 100%;
}
body {
  margin: 0;
}
#app {
  height: 100%;
  display: flex;
  flex-direction: column;
}
.toggle {
  height: 35px;
  line-height: 35px;
  font-size: 14px;
  text-align: center;
}
.toggle label {
  margin-right: 15px;
}
.map-container {
  width: 100%;
  flex: 1;
}
</style>
