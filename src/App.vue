<template>
  <div class="container">
    <toolBar class="tool-bar" @search="onSearchPlace" @mode-change="onModeChange" @length="onDoLength"
      :return-length="totalKiloMeter" />
    <div id="map-container"></div>
  </div>
</template>

<script setup>
import toolBar from './components/toolBar.vue'
import { onMounted, ref } from 'vue'
import L from 'leaflet'
import 'leaflet.heat'
import 'leaflet/dist/leaflet.css'
import anhuiGeoJsonData from './data/anhui.json'
import * as turf from '@turf/turf'
import ChinaGeoJsonData from './data/China.json'

const map = ref(null)
const searchMarkers = ref([])
let recDrawNum = 0
let recList = []
let roiRectLayerList = []

let isDrawMode = false
let turfRoiList = []
let allAnhuiLayers = []
let isSingleViewMode = false
let isLengthMode = true
let lineList = []
let tempPolyLine = null   // 保存当前绘制的折线实例
let totalKiloMeter = ref(0)

let customPoints = []          // 自定义绘制的点序列
let customPreviewLine = null   // 预览折线（橡皮筋效果）
let isCustomDrawing = false    // 是否正在绘制自定义多边形

onMounted(() => {
  function getRandomSoftColor() {
    const hue = Math.floor(Math.random() * 360);
    // saturation 饱和度70%，lightness亮度60%，保证填充不黑不惨白
    return `hsl(${hue}, 70%, 60%)`
  }
  // 加载底图
  const osmLayer = L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', { attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors' })
  const cartoDarkLayer = L.tileLayer('https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}.png', { attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>, &copy; <a href="https://carto.com/attributions">CARTO</a>' })
  // 加载geojson数据
  const geoJsonLayer = L.geoJSON(anhuiGeoJsonData, {
    style: (feature) => {
      const fillColor = getRandomSoftColor()
      return {
        fillColor: fillColor,    // 填充随机柔和色
        fillOpacity: 0.65,       // 透明度，不要太实挡住底图
        color: '#222',           // 边界线固定深黑色，区分地块
        weight: 1.2,             // 边界线粗细
        stroke: true
      }
    },
    onEachFeature: (feature, layer) => {
      allAnhuiLayers.push({ feature, layer })
      layer._originStyle = {
        fillColor: layer.options.fillColor,
        fillOpacity: layer.options.fillOpacity,
        color: layer.options.color,
        weight: layer.options.weight,
        stroke: layer.options.stroke
      }

      layer.on('mouseover', () => {
        if (isSingleViewMode) return
        layer.setStyle({
          fillOpacity: 0.92,
          weight: 2.5
        })
      })

      layer.on('mouseout', () => {
        if (isSingleViewMode) return
        // 直接使用保存好的原始完整样式对象恢复
        layer.setStyle(layer._originStyle)
      })

      layer.on('dblclick', (e) => {
        L.DomEvent.stopPropagation(e) //阻止地图本身双击放大
        isSingleViewMode = true
        map.value.fitBounds(layer.getBounds())
      }
      )
      const area = turf.area(feature)
      const areaKm2 = (area / 1e6).toFixed(2)
      layer.bindPopup(`<b>${feature.properties.name}</b><br>中心坐标：${feature.properties.center[0]}, ${feature.properties.center[1]}<br>面积：${areaKm2} km²`)
    }
  })
  const ChinaGeoJsonLayer = L.geoJSON(ChinaGeoJsonData)
  // 加载热力图

  // 创建地图
  map.value = L.map('map-container', { layers: [osmLayer, ChinaGeoJsonLayer] })
  map.value.setView([31.86, 117.27], 5)
  const baseMaps = {
    "街道图": osmLayer,
    "暗色图": cartoDarkLayer
  }
  const overlayMaps = {
    "中国及其省级行政区": ChinaGeoJsonLayer,
    "安徽省及其地级市": geoJsonLayer
  }
  L.control.layers(baseMaps, overlayMaps, { position: 'bottomleft', collapsed: true }).addTo(map.value)
})

async function onSearchPlace(name) {
  if (!name.trim() || !map.value) return

  const res = await fetch(`https://nominatim.openstreetmap.org/search?q=${name}&format=jsonv2&limit=5`, { headers: { 'Accept-Language': 'zh-CN' } })

  const data = await res.json()
  if (!data.length) return alert('没找到')
  console.log(data)
  // 安徽大学
  clearSearchMarkers()

  const markerGroup = L.featureGroup()

  data.forEach((item, index) => {
    const lat = parseFloat(item.lat)
    const lng = parseFloat(item.lon)
    const marker = L.marker([lat, lng])
    marker.bindPopup(`<b>${index + 1}. ${name}</b><br>类型：${item.type}<br>坐标：${lat.toFixed(4)},${lng.toFixed(4)}`)

    markerGroup.addLayer(marker)
  })
  markerGroup.addTo(map.value)
  searchMarkers.value.push(markerGroup)

  map.value.fitBounds(markerGroup.getBounds(), { padding: [50, 50] })
}

function onModeChange(mode, payload) {
  const drawType = payload.drawType || null
  if (mode === 'select' && payload.run) {
    console.log("已经进入绘制模式，请在地图上按下鼠标！")
    isDrawMode = true
    if (drawType === 'rect') {
      recDrawNum = 0
      recList = []
      map.value.on('mousedown', onMapMouseDown)
    } else if (drawType === 'custom') {
      customPoints = []
      map.value.on('mousedown', onMapCustomMouseDown)
      map.value.on('dblclick', onMapCustomDblClick)
      // 阻止双击放大地图（和你在 geoJsonLayer 里一样的处理）
      map.value.doubleClickZoom.disable()
    }
  }

  if (mode === 'select' && payload.finish) {
    isDrawMode = false
    if (drawType === 'rect') {
      recDrawNum = 0
      recList = []
      map.value.off('mousedown', onMapMouseDown)
    } else if (drawType === 'custom') {
      // 如果还在绘制中直接点"完成"，自动结束当前多边形
      if (customPoints.length >= 3) {
        onMapCustomDblClick({ latlng: customPoints[customPoints.length - 1] })
      }
      map.value.off('mousedown', onMapCustomMouseDown)
      map.value.off('dblclick', onMapCustomDblClick)
      map.value.doubleClickZoom.enable()
      _resetCustomDraw()
    }
    console.log('点击完成，全部ROI‑GeoJSON数组：', allRoiGeoJson)
  }

  if (payload.clear) {
    isDrawMode = false
    recDrawNum = 0
    recList = []
    customPoints = []
    console.log("0")
    if (customPreviewLine && map.value) {
      map.value.removeLayer(customPreviewLine)
      customPreviewLine = null
      console.log("1")
    }

    roiRectLayerList.forEach(layer => {
      map.value.removeLayer(layer)
    })
    roiRectLayerList = []
    turfRoiList = []

    map.value.off('mousedown', onMapMouseDown)
    map.value.off('mousedown', onMapCustomMouseDown)
    map.value.off('dblclick', onMapCustomDblClick)
    map.value.doubleClickZoom.enable()

  }

}

function onDoLength(mode) {
  if (mode.start) {
    console.log("已经进入绘制模式，请在地图上按下鼠标！")
    isLengthMode = true
    lineList = []
    if (map.value) {
      map.value.off('mousedown', onMapLineMouseDown)
      map.value.on('mousedown', onMapLineMouseDown)
    }
  } else if (!mode.start || mode.clear) {
    isLengthMode = false
    lineList = []
    if (tempPolyLine && map.value) {
      map.value.removeLayer(tempPolyLine)
    }
    tempPolyLine = null
    if (map.value) {
      // ✅on 和 off 使用同一个函数 onMapLineMouseDown
      map.value.off('mousedown', onMapLineMouseDown)
    }
  }
}

function onMapMouseDown(e) {
  if (!isDrawMode) return
  const latlng = e.latlng
  const lat = latlng.lat
  const lng = latlng.lng
  recList.push([lat, lng])
  recDrawNum++
  console.log(`已选点${recDrawNum}：`, [lat, lng])
  if (recDrawNum >= 2) {
    const p1 = recList[0]
    const p2 = recList[1]

    const roiRectLayer = L.rectangle([p1, p2], {
      color: '#1a73e8',
      weight: 2,
      fillColor: '#1a73e8',
      fillOpacity: 0.12
    }).addTo(map.value)
    roiRectLayerList.push(roiRectLayer)
    const bounds = roiRectLayer.getBounds()
    // 转变为geoJson-feature
    const turfRoi = turf.bboxPolygon([
      bounds.getWest(),
      bounds.getSouth(),
      bounds.getEast(),
      bounds.getNorth()
    ])
    console.log('生成单个turfRoi', turfRoi)
    turfRoiList.push(turfRoi)
    recDrawNum = 0
    recList = []
  }
}

function onMapLineMouseDown(e) {
  if (!isLengthMode) return
  const latlng = e.latlng
  const lat = latlng.lat
  const lng = latlng.lng
  lineList.push([lat, lng])
  console.log(`已选点：`, [lat, lng])
  if (lineList.length >= 2) {
    if (!tempPolyLine) {
      // 第一次够2个点：创建并添加到地图
      tempPolyLine = L.polyline(lineList, {
        color: 'red',
        weight: 3
      }).addTo(map.value)
    } else {
      // 后续点击：只更新点位，折线自动延长
      tempPolyLine.setLatLngs(lineList)
    }
    totalKiloMeter.value = (calcPolyLength(lineList) / 1e3).toFixed(2)
    console.log('总长度(千米)', totalKiloMeter.value)
  }
}

function onMapCustomMouseDown(e) {
  if (!isDrawMode || selectDrawType.value !== 'custom') return

  const latlng = e.latlng
  customPoints.push([latlng.lat, lng.latlng.lng])
  console.log(`自定义绘制 - 已选点 ${customPoints.length}：`, [latlng.lat, latlng.lng])

  // 实时更新预览线
  if (!customPreviewLine) {
    customPreviewLine = L.polyline(customPoints, {
      color: '#e8710a',
      weight: 2.5,
      dashArray: '6, 6',
      opacity: 0.8
    }).addTo(map.value)
  } else {
    customPreviewLine.setLatLngs(customPoints)
  }
}

// 双击结束绘制
function onMapCustomDblClick(e) {
  if (!isDrawMode || selectDrawType.value !== 'custom') return
  if (customPoints.length < 3) {
    alert('自定义边框至少需要 3 个点才能构成多边形')
    return
  }

  // 闭合多边形（turf.polygon 要求首尾点相同）
  const closedCoords = [...customPoints, customPoints[0]]

  // 创建最终多边形图层
  const polygonLayer = L.polygon(customPoints, {
    color: '#e8710a',
    weight: 2,
    fillColor: '#e8710a',
    fillOpacity: 0.15
  }).addTo(map.value)

  roiRectLayerList.push(polygonLayer) // 复用清除逻辑

  // 生成 turf polygon
  const turfPolygon = turf.polygon([closedCoords])
  console.log('生成自定义turfPolygon', turfPolygon)
  turfRoiList.push(turfPolygon)

  // 清理绘制态
  _resetCustomDraw()
}

function _resetCustomDraw() {
  customPoints = []
  if (customPreviewLine && map.value) {
    map.value.removeLayer(customPreviewLine)
  }
  customPreviewLine = null
}

function clearSearchMarkers() {
  searchMarkers.value.forEach(marker => {
    map.value.removeLayer(marker)
  })
  searchMarkers.value = []
}

function calcPolyLength(polyLatLngs) {
  let total = 0
  for (let i = 0; i < polyLatLngs.length - 1; i++) {
    const p1 = L.latLng(polyLatLngs[i])
    const p2 = L.latLng(polyLatLngs[i + 1])
    total += p1.distanceTo(p2)
  }
  return total
}
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'PingFang SC', 'Hiragino Sans GB', 'Microsoft YaHei', sans-serif;
  -webkit-font-smoothing: antialiased;
}

.container {
  width: 100vw;
  height: 100vh;
  position: relative;
  background-color: #e8eaed;
  overflow: hidden;
}

#map-container {
  height: 100%;
  width: 100%;
}

/* 工具栏定位 */
.tool-bar {
  position: fixed;
  top: 16px;
  left: 50%;
  transform: translateX(-50%);
  width: 480px;
  max-width: calc(100vw - 32px);
  z-index: 999;
}

/* 底部信息条 */
.map-footer {
  position: fixed;
  bottom: 12px;
  right: 16px;
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 14px;
  background: rgba(255, 255, 255, 0.85);
  backdrop-filter: blur(8px);
  border-radius: 20px;
  font-size: 12px;
  color: #5f6368;
  z-index: 999;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.6);
}

.footer-divider {
  color: #dadce0;
}

.footer-item {
  white-space: nowrap;
}
</style>