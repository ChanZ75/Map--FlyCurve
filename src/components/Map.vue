<template>
  <div id="map"></div>
</template>
<script setup>
import * as maptalks from "maptalks";
import * as maptalksThree from "maptalks.three";
import * as maptalksGl from "@maptalks/gl";
import * as THREE from "three";
import { ref, onMounted, onBeforeUnmount } from "vue";
import sichuan from "./json/sichuan.json";
import scPoint from "./json/sichuan_point.json";
import scBorder from "./json/sichuan-border.json";
import { createFlyCurve, timerFlyCurve } from "../flyCurve.js";

const map = ref(null);
onMounted(() => {
  // 初始化地图
  map.value = new maptalks.Map("map", {
    center: [102.839172, 30.273837],
    zoom: 6.5,
    pitch: 42.79999999999998,
    bearing: 0.9000000000005457,
    dragRotate: false,
    dragPitch: false,
    centerCross: false,
    doubleClickZoom: false,
    baseLayer: new maptalks.TileLayer("base", {
      urlTemplate:
        "https://server.arcgisonline.com/arcgis/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}",
      cssFilter: "grayscale(100%)",
    }),
  });

  groupLayer.addTo(map.value);
  addPoints(scPoint);
  addLine(scBorder);
  marker.addTo(map.value);
});

onBeforeUnmount(() => {
  // 流光效果js文件中的定时器，为了避免内存泄漏问题，最好在销毁前清除
  clearInterval(timerFlyCurve);
});

let linePoints = [];
const timer = ref(null);

var threeLayer = new maptalksThree.ThreeLayer("t", {
  forceRenderOnMoving: true,
  forceRenderOnRotating: true,
  identifyCountOnEvent: 1,
});
threeLayer.prepareToDraw = function (gl, scene, camera) {
  var light = new THREE.AmbientLight(0xffffff);
  light.position.set(0, -10, 10).normalize();
  scene.add(light);

  scene.add(new THREE.AmbientLight(0xffffff, 0.2));
  camera.add(new THREE.AmbientLight(0xffffff, 1));
  addExtrudePolygon(sichuan);
  addFlyCurveLine(scene, linePoints);
  animation();
};
const pointLayer = new maptalks.VectorLayer("pointLayer", [], {
  altitude: 20005,
});

const sceneConfig = {
  postProcess: {
    enable: true,
    antialias: { enable: true },
    bloom: {
      enable: true,
      factor: 0.3, // 辉光强度
    },
  },
};
const groupLayer = new maptalksGl.GroupGLLayer("group", [threeLayer], {
  sceneConfig,
});

const extrudepolygons = [];
const extrudelines = [];
//默认材质
const material = new THREE.MeshBasicMaterial({
  transparent: true,
  opacity: 0.1,
  color: "#2987bc",
});
//高光材质
const highlightmaterial = new THREE.MeshPhongMaterial({
  transparent: true,
  opacity: 0.1,
  color: "#43d3ce",
});
// 边界线材质
const lineMaterial = new THREE.LineBasicMaterial({
  linewidth: 2,
  color: "rgba(31,241,245)",
  transparent: true,
  opacity: 1,
});

function addExtrudePolygon(geojson) {
  geojson.features.slice(0, Infinity).forEach((f) => {
    const lineStrings = [];
    const geometry = f.geometry;
    if (geometry.type === "MultiPolygon") {
      geometry.coordinates.forEach((coordinates) => {
        lineStrings.push(new maptalks.LineString(coordinates[0]));
      });
    } else {
      lineStrings.push(new maptalks.LineString(geometry.coordinates[0]));
    }

    const polygon = threeLayer.toExtrudePolygon(
      f,
      {
        height: 20000,
        interactive: true,
        bloom: true,
        attributes: f.properties,
      },
      material
    );
    polygon.on("mouseover mouseout click", function (e) {
      switch (e.type) {
        case "mouseout":
          if (
            marker._markerCoord.x !== this.options.attributes.coordinates[0] &&
            marker._markerCoord.y !== this.options.attributes.coordinates[1]
          ) {
            this.setSymbol(material);
          }
          break;
        case "mouseover":
          map.value.setCursor("pointer");
          this.setSymbol(highlightmaterial);
          break;
        case "click":
          //若当前存在高亮polygon 先将其取消高亮
          if (highlightIndex != -1) {
            extrudepolygons[highlightIndex].setSymbol(material);
          }
          highlightIndex = extrudepolygons.indexOf(this) - 1;
          startTimer(5000);
      }
    });
    extrudepolygons.push(polygon);
    lineStrings.forEach((lineString) => {
      const line = threeLayer.toLine(
        lineString,
        { altitude: 20000 * 1.0001, interactive: false, bloom: true },
        lineMaterial
      );
      extrudelines.push(line);
    });
  });

  threeLayer.addMesh(extrudepolygons);
  threeLayer.addMesh(extrudelines);
  startTimer(5000);
}

//加载四川省边界线 作为流光轨迹
function addLine(jsondata) {
  // 遍历省份构建模型
  jsondata.features.forEach((elem) => {
    // 新建一个省份容器：用来存放省份对应的模型和轮廓线
    const coordinates = elem.geometry.coordinates;
    coordinates.forEach((multiPolygon) => {
      multiPolygon.forEach((polygon) => {
        const positions = [];
        for (let i = 0; i < polygon.length; i++) {
          var pos = threeLayer.coordinateToVector3(
            [polygon[i][0], polygon[i][1]],
            320 //流光高度
          );
          positions.push(pos.x, pos.y, pos.z);
          linePoints.push(pos);
        }
      });
    });
    const lineString = new maptalks.LineString(coordinates[0][0]);
    const polyline = threeLayer.toExtrudeLine(
      lineString,
      { height: 20000, interactive: false, bloom: true },
      lineMaterial
    );
    extrudelines.push(polyline);
  });
}

//添加流光
function addFlyCurveLine(scene, points) {
  var closed = true;
  var flyLine = createFlyCurve(points, closed);
  scene.add(flyLine);
}

function animation() {
  threeLayer._needsUpdate = !threeLayer._needsUpdate;
  if (threeLayer._needsUpdate) {
    threeLayer.redraw();
  }
  requestAnimationFrame(animation);
}
// 加载地名点
function addPoints(points) {
  pointLayer.setStyle([
    {
      filter: ["!=", "name", ""],
      symbol: [
        {
          textName: "{name}",
          textSize: 12,
          textFill: "#fff",
          textHaloFill: "rgba(0,0,0,0.3)",
          textHaloRadius: 2,
        },
      ],
    },
  ]);
  points.features.forEach((feature) => {
    maptalks.GeoJSON.toGeometry(feature).addTo(pointLayer);
  });
  pointLayer.addTo(map.value);
}

let dom = document.createElement("div");
dom.innerHTML = "";
dom.className = "online";
//添加marker
let marker = new maptalks.ui.UIMarker(
  [102.663822040699785, 29.912182469248506],
  {
    draggable: false,
    content: dom,
    dy: -50,
    altitude: 20000,
  }
);

let highlightIndex = -1; // 高亮索引 初始值为-1
//按顺序mark所有城市
function markCity() {
  if (highlightIndex != -1) {
    extrudepolygons[highlightIndex].setSymbol(material);
  }
  highlightIndex++;
  //当mark到最后一个城市后 重头开始
  if (highlightIndex == extrudepolygons.length) {
    highlightIndex = 0;
  }
  const currentPolygon = extrudepolygons[highlightIndex];
  const currentCoords = currentPolygon.options.attributes.coordinates;
  currentCoords &&
    marker.setCoordinates(new maptalks.Coordinate(currentCoords));
  currentPolygon.setSymbol(highlightmaterial);
}

function startTimer(time) {
  clearTimer();
  markCity();
  timer.value = setInterval(markCity, time);
}

function clearTimer() {
  if (timer.value) clearInterval(timer.value);
  timer.value = null;
}
</script>
<style>
html,
body {
  margin: 0px;
  height: 100%;
  width: 100%;
}

#map {
  width: 100%;
  height: 100%;
  background-color: #000;
}

.online {
  width: 186px;
  height: 154px;
  background: url(../assets/online-frames/online_01.png),
    url(../assets/online-frames/online_02.png),
    url(../assets/online-frames/online_03.png),
    url(../assets/online-frames/online_04.png),
    url(../assets/online-frames/online_05.png),
    url(../assets/online-frames/online_06.png),
    url(../assets/online-frames/online_07.png),
    url(../assets/online-frames/online_08.png),
    url(../assets/online-frames/online_09.png),
    url(../assets/online-frames/online_10.png),
    url(../assets/online-frames/online_11.png),
    url(../assets/online-frames/online_12.png),
    url(../assets/online-frames/online_13.png),
    url(../assets/online-frames/online_14.png);
  position: relative;
  animation: onlineFrames 2.5s linear infinite;
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
}

@keyframes onlineFrames {
  0% {
    background: url(../assets/online-frames/online_01.png) no-repeat;
  }
  8% {
    background: url(../assets/online-frames/online_02.png) no-repeat;
  }
  15% {
    background: url(../assets/online-frames/online_03.png) no-repeat;
  }
  23% {
    background: url(../assets/online-frames/online_04.png) no-repeat;
  }
  31% {
    background: url(../assets/online-frames/online_05.png) no-repeat;
  }
  38% {
    background: url(../assets/online-frames/online_06.png) no-repeat;
  }
  46% {
    background: url(../assets/online-frames/online_07.png) no-repeat;
  }
  54% {
    background: url(../assets/online-frames/online_08.png) no-repeat;
  }
  62% {
    background: url(../assets/online-frames/online_09.png) no-repeat;
  }
  69% {
    background: url(../assets/online-frames/online_10.png) no-repeat;
  }
  77% {
    background: url(../assets/online-frames/online_11.png) no-repeat;
  }
  84% {
    background: url(../assets/online-frames/online_12.png) no-repeat;
  }
  92% {
    background: url(../assets/online-frames/online_13.png) no-repeat;
  }
  100% {
    background: url(../assets/online-frames/online_14.png) no-repeat;
  }
}
</style>
