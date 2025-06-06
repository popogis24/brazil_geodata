<template>
  <div class="control-panel">
    <h3>Controle de Camadas</h3>
    
    <!-- Camadas Base -->
    <div class="layer-section">
      <h4>Camadas Base</h4>
      <div v-for="layer in baseLayers" :key="layer.id" class="layer-item">
        <input 
          type="radio" 
          :id="layer.id" 
          :value="layer.id"
          v-model="selectedBaseLayer"
          @change="toggleLayer(layer.id)"
          class="layer-checkbox"
        />
        <label :for="layer.id" class="layer-name">{{ layer.name }}</label>
      </div>
    </div>

    <!-- Camadas Overlay -->
    <div class="layer-section">
      <h4>Camadas Overlay</h4>
      <div v-for="layer in overlayLayers" :key="layer.id" class="layer-item">
        <input 
          type="checkbox" 
          :id="layer.id" 
          :checked="layer.visible"
          @change="toggleLayer(layer.id)"
          class="layer-checkbox"
        />
        <label :for="layer.id" class="layer-name">{{ layer.name }}</label>
        <button 
          v-if="layer.removable" 
          @click="removeLayer(layer.id)"
          class="btn btn-danger btn-sm"
          title="Remover camada"
        >
          ✕
        </button>
      </div>
    </div>

    <!-- Camadas GeoJSON -->
    <div v-if="geojsonLayers.length > 0" class="layer-section">
      <h4>Camadas GeoJSON</h4>
      <div v-for="layer in geojsonLayers" :key="layer.id" class="layer-item">
        <input 
          type="checkbox" 
          :id="layer.id" 
          :checked="layer.visible"
          @change="toggleLayer(layer.id)"
          class="layer-checkbox"
        />
        <label :for="layer.id" class="layer-name">{{ layer.name }}</label>
        <button 
          @click="removeLayer(layer.id)"
          class="btn btn-danger btn-sm"
          title="Remover camada"
        >
          ✕
        </button>
      </div>
    </div>

    <!-- Informações da Camada -->
    <div v-if="selectedLayerInfo" class="layer-info">
      <h4>Informações da Camada</h4>
      <p><strong>Nome:</strong> {{ selectedLayerInfo.name }}</p>
      <p><strong>Tipo:</strong> {{ getLayerTypeLabel(selectedLayerInfo.type) }}</p>
      <p v-if="selectedLayerInfo.attribution">
        <strong>Fonte:</strong> {{ selectedLayerInfo.attribution }}
      </p>
    </div>
  </div>
</template>

<script>
export default {
  name: 'LayerControl',
  props: {
    layers: {
      type: Array,
      required: true
    }
  },
  data() {
    return {
      selectedLayerInfo: null
    }
  },
  computed: {
    baseLayers() {
      return this.layers.filter(layer => layer.type === 'base')
    },
    overlayLayers() {
      return this.layers.filter(layer => 
        layer.type !== 'base' && layer.type !== 'geojson'
      )
    },
    geojsonLayers() {
      return this.layers.filter(layer => layer.type === 'geojson')
    },
    selectedBaseLayer: {
      get() {
        const baseLayer = this.baseLayers.find(layer => layer.visible)
        return baseLayer ? baseLayer.id : null
      },
      set(layerId) {
        // Este setter é necessário para o v-model funcionar
        // A lógica real está no método toggleLayer
      }
    }
  },
  methods: {
    toggleLayer(layerId) {
      this.$emit('toggle-layer', layerId)
      
      // Atualiza informações da camada selecionada
      const layer = this.layers.find(l => l.id === layerId)
      if (layer && layer.visible) {
        this.selectedLayerInfo = layer
      }
    },
    
    removeLayer(layerId) {
      if (confirm('Tem certeza que deseja remover esta camada?')) {
        this.$emit('remove-layer', layerId)
        
        // Limpa informações se a camada removida estava selecionada
        if (this.selectedLayerInfo && this.selectedLayerInfo.id === layerId) {
          this.selectedLayerInfo = null
        }
      }
    },
    
    getLayerTypeLabel(type) {
      const typeLabels = {
        'base': 'Camada Base',
        'wms': 'WMS',
        'wfs': 'WFS',
        'geojson': 'GeoJSON',
        'tile': 'Tile'
      }
      return typeLabels[type] || type.toUpperCase()
    },
    
    showLayerInfo(layer) {
      this.selectedLayerInfo = layer
    }
  }
}
</script>

<style scoped>
.layer-section {
  margin-bottom: 1rem;
  border-bottom: 1px solid #eee;
  padding-bottom: 0.5rem;
}

.layer-section h4 {
  font-size: 0.9rem;
  color: #666;
  margin-bottom: 0.5rem;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.layer-item {
  display: flex;
  align-items: center;
  padding: 0.3rem 0;
  gap: 0.5rem;
}

.layer-name {
  flex: 1;
  font-size: 0.85rem;
  cursor: pointer;
  user-select: none;
}

.layer-checkbox {
  margin: 0;
}

.btn-sm {
  padding: 0.2rem 0.4rem;
  font-size: 0.7rem;
  min-width: auto;
  height: auto;
}

.layer-info {
  background-color: #f8f9fa;
  padding: 0.75rem;
  border-radius: 4px;
  margin-top: 1rem;
  border: 1px solid #e9ecef;
}

.layer-info h4 {
  margin-bottom: 0.5rem;
  color: #495057;
  font-size: 0.9rem;
}

.layer-info p {
  margin: 0.25rem 0;
  font-size: 0.8rem;
  color: #6c757d;
}

.layer-info strong {
  color: #495057;
}
</style>