<template>
  <div class="map-container" id="wMap"></div>
  <span class="curZoom" style="position: absolute; bottom: 20px; left: 180px;">{{currentZoom}}</span>
</template>

<script setup>
import {ref, onMounted, watch} from "vue";
import { Map, View } from "ol";
import { Tile as TileLayer, Vector as VectorLayer} from "ol/layer";
import { Vector as VectorSource} from "ol/source";
import {ScaleLine} from 'ol/control';
import Feature from 'ol/Feature';
import { Point } from 'ol/geom';
import { getVectorContext } from "ol/render";
import {Fill, Style, Stroke, Circle} from 'ol/style';
import "ol/ol.css";
import EsriJSON from 'ol/format/EsriJSON';
import WMTS, {optionsFromCapabilities} from 'ol/source/WMTS';
import WMTSCapabilities from 'ol/format/WMTSCapabilities';

const huaiNanUrl = 'http://60.174.203.118:6080/arcgis/rest/services/bengbufenzhongxin/%E6%B7%AE%E5%8D%97%E5%B8%82/MapServer/WMTS/1.0.0/WMTSCapabilities.xml';
const chuZhouUrl = 'http://60.174.203.118:6080/arcgis/rest/services/ChuZhou/ChuZhouFZXDT/MapServer/WMTS/1.0.0/WMTSCapabilities.xml';

let map, mapView;

let isClick = ref(false)
let currentZoom = ref(7);
let loadCount = ref(0);

onMounted(() => {
  initMap();
})

const initMap = () => {
  mapView = new View({
    enableRotation: false,
    projection: "EPSG:4326",
    center: [117.621597, 32.452257],
    minZoom: 8,
    zoom: 9
  })
  map = new Map({
    target: 'wMap',
    controls: [],
    layers: [],
    view: mapView
  })
  map.addControl(new ScaleLine());
  // 添加滁州图层
  fetch(chuZhouUrl)
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
    console.log(options);
    map.addLayer(
      new TileLayer({
        source: new WMTS(options),
      })
    );
  });

  // 添加淮南图层
  fetch(huaiNanUrl)
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
    console.log(options);
    map.addLayer(
      new TileLayer({
        source: new WMTS(options),
      })
    );
  });
  // 监听
  mapView.on("change:resolution", e => {
    currentZoom.value = mapView.getZoom().toFixed(1);
  })

  // 点击事件，cleTm防止多次执行相同操作
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
      // 200ms后把isClick重新改为false，不然点第二个点会无反应，被上面return掉(103行)
      cleTm = setTimeout(() => {
        isClick.value = false;
      }, 200);
      // 获取当前点击的feature
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
  // 最中间圆点保持不动，让r1和r2想歪扩散
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

  // 让点闪烁起来，上一步仅绘制了图层，并不会闪
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
        // 调用layer的changed方法，进行刷新
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
}
.map-container {
  width: 100%;
  height: 100%;
}
</style>
