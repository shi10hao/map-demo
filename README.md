安徽省所有985，211学校的坐标以及geojson，还有所有一本以及二本，专科的坐标，分布，对应高考录取人数以及高考平均分。
两点之间的距离
OSM:
'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', { attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors' }

cartoDark:
'https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}.png', { attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>, &copy; <a href="https://carto.com/attributions">CARTO</a>' }

nominatim:
`https://nominatim.openstreetmap.org/search?q=${name}&format=jsonv2&limit=5`, { headers: { 'Accept-Language': 'zh-CN' } }

  const ChinaGeoJsonLayer = L.geoJSON(ChinaGeoJsonData, {
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
      layer._originStyle = {
        fillColor: layer.options.fillColor,
        fillOpacity: layer.options.fillOpacity,
        color: layer.options.color,
        weight: layer.options.weight,
        stroke: layer.options.stroke
      }
      layer.on('mouseover', () => {
        layer.setStyle({
          fillOpacity: 0.92,
          weight: 2.5
        })
      })
      layer.on('mouseout', () => {
        // 直接使用保存好的原始完整样式对象恢复
        layer.setStyle(layer._originStyle)
      })
      const area = turf.area(feature)
      const areaKm2 = (area / 1e6).toFixed(2)
      layer.bindPopup(`<b>${feature.properties.name}</b><br>中心坐标：${feature.properties.center[0]}, ${feature.properties.center[1]}<br>面积：${areaKm2} km²`)
    }
  })
