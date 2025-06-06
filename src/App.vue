<template>
  <div id="app">
    <header class="header">
      <h1>Brazil GeoData Viewer</h1>
      <p>Sistema de visualização e análise de dados geoespaciais</p>
    </header>
    
    <div class="main-container">
      <div class="sidebar">
        <LayerControl 
          :layers="layers"
          @toggle-layer="toggleLayer"
          @remove-layer="removeLayer"
        />
        
        <UploadGeoJSON 
          @geojson-uploaded="addGeoJSONLayer"
        />
        
        <div class="control-panel">
          <h3>Ferramentas de Desenho</h3>
          <button class="btn" @click="toggleDrawing('marker')">
            📍 Ponto
          </button>
          <button class="btn" @click="toggleDrawing('polyline')">
            📏 Linha
          </button>
          <button class="btn" @click="toggleDrawing('polygon')">
            🔷 Polígono
          </button>
          <button class="btn btn-secondary" @click="clearDrawings">
            🗑️ Limpar Desenhos
          </button>
          <button class="btn btn-secondary" @click="exportDrawings">
            💾 Exportar GeoJSON
          </button>
        </div>
        
        <div class="control-panel">
          <h3>Medições</h3>
          <button class="btn" @click="toggleMeasurement('distance')">
            📐 Distância
          </button>
          <button class="btn" @click="toggleMeasurement('area')">
            📊 Área
          </button>
          <div v-if="measurementResult" class="measurement-info">
            {{ measurementResult }}
          </div>
        </div>
      </div>
      
      <div class="map-container">
        <MapView 
          ref="mapView"
          :layers="layers"
          @measurement-result="updateMeasurement"
          @drawing-created="onDrawingCreated"
        />
      </div>
    </div>
  </div>
</template>

<script>
import MapView from './components/MapView.vue'
import LayerControl from './components/LayerControl.vue'
import UploadGeoJSON from './components/UploadGeoJSON.vue'

export default {
  name: 'App',
  components: {
    MapView,
    LayerControl,
    UploadGeoJSON
  },
  data() {
    return {
      layers: [
        {
          id: 'osm',
          name: 'OpenStreetMap',
          type: 'base',
          visible: true,
          url: 'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
          attribution: '© OpenStreetMap contributors'
        },
        {
          id: 'satellite',
          name: 'Satellite',
          type: 'base',
          visible: false,
          url: 'https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}',
          attribution: 'Esri, DigitalGlobe, GeoEye, Earthstar Geographics, CNES/Airbus DS, USDA, USGS, AeroGRID, IGN, and the GIS User Community'
        },
        // Exemplo de camadas WMS do GeoServer
        {
          id: 'brazil_states',
          name: 'Estados do Brasil',
          type: 'wms',
          visible: false,
          url: 'https://geoservicos.ibge.gov.br/geoserver/wms',
          layers: 'CGEO:bc250_unidade_federacao_a',
          format: 'image/png',
          transparent: true,
          attribution: 'IBGE'
        },
        {
          id: 'brazil_municipalities',
          name: 'Municípios do Brasil',
          type: 'wms',
          visible: false,
          url: 'https://geoservicos.ibge.gov.br/geoserver/wms',
          layers: 'CGEO:bc250_municipio_a',
          format: 'image/png',
          transparent: true,
          attribution: 'IBGE'
        }
      ],
      measurementResult: null,
      drawnItems: []
    }
  },
  methods: {
    toggleLayer(layerId) {
      const layer = this.layers.find(l => l.id === layerId)
      if (layer) {
        if (layer.type === 'base') {
          // Para camadas base, desativa todas as outras camadas base
          this.layers.forEach(l => {
            if (l.type === 'base') {
              l.visible = l.id === layerId
            }
          })
        } else {
          layer.visible = !layer.visible
        }
        this.$refs.mapView.updateLayers()
      }
    },
    
    removeLayer(layerId) {
      const index = this.layers.findIndex(l => l.id === layerId)
      if (index > -1 && this.layers[index].type !== 'base') {
        this.layers.splice(index, 1)
        this.$refs.mapView.updateLayers()
      }
    },
    
    addGeoJSONLayer(geojsonData, fileName) {
      const newLayer = {
        id: `geojson_${Date.now()}`,
        name: fileName || 'GeoJSON Layer',
        type: 'geojson',
        visible: true,
        data: geojsonData,
        removable: true
      }
      this.layers.push(newLayer)
      this.$refs.mapView.updateLayers()
    },
    
    toggleDrawing(type) {
      this.$refs.mapView.toggleDrawing(type)
    },
    
    clearDrawings() {
      this.$refs.mapView.clearDrawings()
      this.drawnItems = []
    },
    
    exportDrawings() {
      const drawings = this.$refs.mapView.getDrawings()
      if (drawings.length === 0) {
        alert('Nenhum desenho para exportar')
        return
      }
      
      const geojson = {
        type: 'FeatureCollection',
        features: drawings
      }
      
      const blob = new Blob([JSON.stringify(geojson, null, 2)], {
        type: 'application/json'
      })
      const url = URL.createObjectURL(blob)
      const a = document.createElement('a')
      a.href = url
      a.download = `drawings_${new Date().toISOString().split('T')[0]}.geojson`
      document.body.appendChild(a)
      a.click()
      document.body.removeChild(a)
      URL.revokeObjectURL(url)
    },
    
    toggleMeasurement(type) {
      this.$refs.mapView.toggleMeasurement(type)
    },
    
    updateMeasurement(result) {
      this.measurementResult = result
    },
    
    onDrawingCreated(drawing) {
      this.drawnItems.push(drawing)
    }
  }
}
</script>