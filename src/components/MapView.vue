<template>
  <div id="map" class="leaflet-container"></div>
</template>

<script>
import L from 'leaflet'
import 'leaflet-draw'
import * as turf from '@turf/turf'

// Fix for default markers in Leaflet
delete L.Icon.Default.prototype._getIconUrl
L.Icon.Default.mergeOptions({
  iconRetinaUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/images/marker-icon-2x.png',
  iconUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/images/marker-icon.png',
  shadowUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/images/marker-shadow.png',
})

export default {
  name: 'MapView',
  props: {
    layers: {
      type: Array,
      required: true
    }
  },
  data() {
    return {
      map: null,
      layerInstances: new Map(),
      drawnItems: null,
      drawControl: null,
      currentDrawingMode: null,
      measurementMode: null,
      measurementLayer: null,
      miniMap: null
    }
  },
  mounted() {
    this.initMap()
    this.setupDrawing()
    this.setupMiniMap()
    this.updateLayers()
  },
  beforeUnmount() {
    if (this.map) {
      this.map.remove()
    }
  },
  methods: {
    initMap() {
      // Inicializa o mapa centrado no Brasil
      this.map = L.map('map', {
        center: [-14.235, -51.9253], // Centro do Brasil
        zoom: 4,
        zoomControl: true
      })

      // Adiciona controles de zoom
      this.map.zoomControl.setPosition('topleft')

      // Adiciona controle de escala
      L.control.scale({
        position: 'bottomright',
        metric: true,
        imperial: false
      }).addTo(this.map)
    },

    setupDrawing() {
      // Cria grupo para itens desenhados
      this.drawnItems = new L.FeatureGroup()
      this.map.addLayer(this.drawnItems)

      // Configura controles de desenho
      this.drawControl = new L.Control.Draw({
        position: 'topright',
        draw: {
          polygon: {
            allowIntersection: false,
            drawError: {
              color: '#e1e100',
              message: '<strong>Erro:</strong> As bordas não podem se cruzar!'
            },
            shapeOptions: {
              color: '#97009c'
            }
          },
          polyline: {
            shapeOptions: {
              color: '#f357a1',
              weight: 3
            }
          },
          rect: false,
          circle: false,
          circlemarker: false,
          marker: {
            icon: new L.Icon.Default()
          }
        },
        edit: {
          featureGroup: this.drawnItems,
          remove: true
        }
      })

      // Eventos de desenho
      this.map.on(L.Draw.Event.CREATED, (e) => {
        const layer = e.layer
        const feature = this.layerToGeoJSON(layer)
        
        // Adiciona popup com informações
        this.addPopupToLayer(layer, feature)
        
        this.drawnItems.addLayer(layer)
        this.$emit('drawing-created', feature)
      })

      this.map.on(L.Draw.Event.EDITED, (e) => {
        // Atualiza popups após edição
        e.layers.eachLayer((layer) => {
          const feature = this.layerToGeoJSON(layer)
          this.addPopupToLayer(layer, feature)
        })
      })
    },

    setupMiniMap() {
      // Cria mini mapa para navegação
      const osmMini = L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '© OpenStreetMap contributors'
      })

      this.miniMap = new L.Control.MiniMap(osmMini, {
        position: 'bottomleft',
        width: 150,
        height: 150,
        collapsedWidth: 25,
        collapsedHeight: 25,
        zoomLevelOffset: -5,
        zoomAnimation: false,
        toggleDisplay: true
      }).addTo(this.map)
    },

    updateLayers() {
      // Remove todas as camadas existentes (exceto desenhos)
      this.layerInstances.forEach((layer, id) => {
        if (this.map.hasLayer(layer)) {
          this.map.removeLayer(layer)
        }
      })
      this.layerInstances.clear()

      // Adiciona camadas visíveis
      this.layers.forEach(layerConfig => {
        if (layerConfig.visible) {
          const layer = this.createLayer(layerConfig)
          if (layer) {
            this.layerInstances.set(layerConfig.id, layer)
            this.map.addLayer(layer)
          }
        }
      })
    },

    createLayer(config) {
      switch (config.type) {
        case 'base':
          return L.tileLayer(config.url, {
            attribution: config.attribution,
            maxZoom: 18
          })

        case 'wms':
          return L.tileLayer.wms(config.url, {
            layers: config.layers,
            format: config.format || 'image/png',
            transparent: config.transparent !== false,
            attribution: config.attribution,
            maxZoom: 18
          })

        case 'geojson':
          return L.geoJSON(config.data, {
            style: {
              color: '#3388ff',
              weight: 2,
              opacity: 0.8,
              fillOpacity: 0.3
            },
            onEachFeature: (feature, layer) => {
              // Adiciona popup com propriedades do feature
              if (feature.properties) {
                let popupContent = '<h4>Propriedades:</h4>'
                Object.entries(feature.properties).forEach(([key, value]) => {
                  popupContent += `<strong>${key}:</strong> ${value}<br>`
                })
                layer.bindPopup(popupContent)
              }
            }
          })

        default:
          console.warn(`Tipo de camada não suportado: ${config.type}`)
          return null
      }
    },

    toggleDrawing(type) {
      // Remove controles existentes
      if (this.drawControl) {
        this.map.removeControl(this.drawControl)
      }

      // Configura novo controle baseado no tipo
      const drawOptions = {
        position: 'topright',
        draw: {
          polygon: type === 'polygon',
          polyline: type === 'polyline',
          marker: type === 'marker',
          rect: false,
          circle: false,
          circlemarker: false
        },
        edit: {
          featureGroup: this.drawnItems,
          remove: true
        }
      }

      this.drawControl = new L.Control.Draw(drawOptions)
      this.map.addControl(this.drawControl)
      this.currentDrawingMode = type
    },

    clearDrawings() {
      this.drawnItems.clearLayers()
    },

    getDrawings() {
      const features = []
      this.drawnItems.eachLayer((layer) => {
        features.push(this.layerToGeoJSON(layer))
      })
      return features
    },

    layerToGeoJSON(layer) {
      let feature = layer.toGeoJSON()
      
      // Adiciona propriedades calculadas
      if (feature.geometry.type === 'LineString') {
        const line = turf.lineString(feature.geometry.coordinates)
        const length = turf.length(line, { units: 'kilometers' })
        feature.properties = {
          ...feature.properties,
          length_km: Math.round(length * 100) / 100,
          type: 'Linha'
        }
      } else if (feature.geometry.type === 'Polygon') {
        const polygon = turf.polygon(feature.geometry.coordinates)
        const area = turf.area(polygon) / 1000000 // Convert to km²
        feature.properties = {
          ...feature.properties,
          area_km2: Math.round(area * 100) / 100,
          type: 'Polígono'
        }
      } else if (feature.geometry.type === 'Point') {
        feature.properties = {
          ...feature.properties,
          type: 'Ponto',
          coordinates: feature.geometry.coordinates.join(', ')
        }
      }

      return feature
    },

    addPopupToLayer(layer, feature) {
      let popupContent = `<h4>${feature.properties.type}</h4>`
      
      if (feature.properties.length_km) {
        popupContent += `<strong>Comprimento:</strong> ${feature.properties.length_km} km<br>`
      }
      
      if (feature.properties.area_km2) {
        popupContent += `<strong>Área:</strong> ${feature.properties.area_km2} km²<br>`
      }
      
      if (feature.properties.coordinates) {
        popupContent += `<strong>Coordenadas:</strong> ${feature.properties.coordinates}<br>`
      }

      layer.bindPopup(popupContent)
    },

    toggleMeasurement(type) {
      // Remove camada de medição anterior
      if (this.measurementLayer) {
        this.map.removeLayer(this.measurementLayer)
        this.measurementLayer = null
      }

      if (this.measurementMode === type) {
        this.measurementMode = null
        this.$emit('measurement-result', null)
        return
      }

      this.measurementMode = type
      this.measurementLayer = new L.FeatureGroup().addTo(this.map)

      // Configura eventos de clique para medição
      this.map.on('click', this.handleMeasurementClick)
    },

    handleMeasurementClick(e) {
      if (!this.measurementMode || !this.measurementLayer) return

      const latlng = e.latlng
      
      if (this.measurementMode === 'distance') {
        this.handleDistanceMeasurement(latlng)
      } else if (this.measurementMode === 'area') {
        this.handleAreaMeasurement(latlng)
      }
    },

    handleDistanceMeasurement(latlng) {
      // Adiciona marcador
      const marker = L.marker(latlng).addTo(this.measurementLayer)
      
      const markers = []
      this.measurementLayer.eachLayer(layer => {
        if (layer instanceof L.Marker) {
          markers.push(layer.getLatLng())
        }
      })

      if (markers.length >= 2) {
        // Calcula distância entre os dois últimos pontos
        const point1 = turf.point([markers[markers.length-2].lng, markers[markers.length-2].lat])
        const point2 = turf.point([markers[markers.length-1].lng, markers[markers.length-1].lat])
        const distance = turf.distance(point1, point2, { units: 'kilometers' })
        
        // Desenha linha
        const line = L.polyline([markers[markers.length-2], markers[markers.length-1]], {
          color: 'red',
          weight: 3
        }).addTo(this.measurementLayer)

        this.$emit('measurement-result', `Distância: ${Math.round(distance * 100) / 100} km`)
      }
    },

    handleAreaMeasurement(latlng) {
      // Adiciona marcador
      const marker = L.marker(latlng).addTo(this.measurementLayer)
      
      const markers = []
      this.measurementLayer.eachLayer(layer => {
        if (layer instanceof L.Marker) {
          markers.push(layer.getLatLng())
        }
      })

      if (markers.length >= 3) {
        // Cria polígono e calcula área
        const coordinates = markers.map(m => [m.lng, m.lat])
        coordinates.push(coordinates[0]) // Fecha o polígono
        
        const polygon = turf.polygon([coordinates])
        const area = turf.area(polygon) / 1000000 // Convert to km²
        
        // Desenha polígono
        L.polygon(markers, {
          color: 'blue',
          fillColor: 'blue',
          fillOpacity: 0.3
        }).addTo(this.measurementLayer)

        this.$emit('measurement-result', `Área: ${Math.round(area * 100) / 100} km²`)
      }
    }
  },

  watch: {
    layers: {
      handler() {
        this.updateLayers()
      },
      deep: true
    }
  }
}
</script>

<style scoped>
#map {
  height: 100%;
  width: 100%;
}
</style>