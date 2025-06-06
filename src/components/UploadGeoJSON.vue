<template>
  <div class="control-panel">
    <h3>Upload GeoJSON</h3>
    
    <div 
      class="upload-area"
      :class="{ 'dragover': isDragOver }"
      @drop="handleDrop"
      @dragover="handleDragOver"
      @dragleave="handleDragLeave"
      @click="triggerFileInput"
    >
      <div class="upload-content">
        <div class="upload-icon">📁</div>
        <p>Clique aqui ou arraste um arquivo GeoJSON</p>
        <small>Formatos suportados: .geojson, .json</small>
      </div>
    </div>

    <input 
      ref="fileInput"
      type="file" 
      accept=".geojson,.json"
      @change="handleFileSelect"
      class="file-input"
    />

    <div v-if="isLoading" class="loading-message">
      Carregando arquivo...
    </div>

    <div v-if="errorMessage" class="error-message">
      {{ errorMessage }}
    </div>

    <div v-if="successMessage" class="success-message">
      {{ successMessage }}
    </div>

    <!-- Exemplo de URL para teste -->
    <div class="url-input-section">
      <h4>Ou carregue via URL</h4>
      <div class="url-input-group">
        <input 
          v-model="geoJsonUrl"
          type="url" 
          placeholder="https://exemplo.com/dados.geojson"
          class="url-input"
        />
        <button 
          @click="loadFromUrl"
          :disabled="!geoJsonUrl || isLoading"
          class="btn btn-secondary"
        >
          Carregar
        </button>
      </div>
    </div>

    <!-- Exemplos de dados -->
    <div class="examples-section">
      <h4>Exemplos</h4>
      <button 
        @click="loadExample('brazil-states')"
        class="btn btn-secondary btn-sm"
      >
        Estados do Brasil
      </button>
      <button 
        @click="loadExample('sample-points')"
        class="btn btn-secondary btn-sm"
      >
        Pontos de Exemplo
      </button>
    </div>
  </div>
</template>

<script>
export default {
  name: 'UploadGeoJSON',
  data() {
    return {
      isDragOver: false,
      isLoading: false,
      errorMessage: '',
      successMessage: '',
      geoJsonUrl: ''
    }
  },
  methods: {
    triggerFileInput() {
      this.$refs.fileInput.click()
    },

    handleDragOver(e) {
      e.preventDefault()
      this.isDragOver = true
    },

    handleDragLeave(e) {
      e.preventDefault()
      this.isDragOver = false
    },

    handleDrop(e) {
      e.preventDefault()
      this.isDragOver = false
      
      const files = e.dataTransfer.files
      if (files.length > 0) {
        this.processFile(files[0])
      }
    },

    handleFileSelect(e) {
      const files = e.target.files
      if (files.length > 0) {
        this.processFile(files[0])
      }
    },

    async processFile(file) {
      this.clearMessages()
      
      // Validação do tipo de arquivo
      const validExtensions = ['.geojson', '.json']
      const fileExtension = file.name.toLowerCase().substring(file.name.lastIndexOf('.'))
      
      if (!validExtensions.includes(fileExtension)) {
        this.errorMessage = 'Formato de arquivo não suportado. Use .geojson ou .json'
        return
      }

      // Validação do tamanho (máximo 10MB)
      if (file.size > 10 * 1024 * 1024) {
        this.errorMessage = 'Arquivo muito grande. Tamanho máximo: 10MB'
        return
      }

      this.isLoading = true

      try {
        const text = await this.readFileAsText(file)
        const geojsonData = JSON.parse(text)
        
        // Validação básica do GeoJSON
        if (!this.isValidGeoJSON(geojsonData)) {
          throw new Error('Arquivo não é um GeoJSON válido')
        }

        // Emite evento com os dados
        this.$emit('geojson-uploaded', geojsonData, file.name)
        
        this.successMessage = `Arquivo "${file.name}" carregado com sucesso!`
        
        // Limpa o input
        this.$refs.fileInput.value = ''
        
      } catch (error) {
        this.errorMessage = `Erro ao processar arquivo: ${error.message}`
      } finally {
        this.isLoading = false
      }
    },

    async loadFromUrl() {
      if (!this.geoJsonUrl) return

      this.clearMessages()
      this.isLoading = true

      try {
        const response = await fetch(this.geoJsonUrl)
        
        if (!response.ok) {
          throw new Error(`Erro HTTP: ${response.status}`)
        }

        const geojsonData = await response.json()
        
        if (!this.isValidGeoJSON(geojsonData)) {
          throw new Error('URL não contém um GeoJSON válido')
        }

        // Extrai nome do arquivo da URL
        const fileName = this.geoJsonUrl.split('/').pop() || 'dados-url.geojson'
        
        this.$emit('geojson-uploaded', geojsonData, fileName)
        
        this.successMessage = `Dados carregados da URL com sucesso!`
        this.geoJsonUrl = ''
        
      } catch (error) {
        this.errorMessage = `Erro ao carregar da URL: ${error.message}`
      } finally {
        this.isLoading = false
      }
    },

    loadExample(exampleType) {
      this.clearMessages()
      
      let exampleData
      let fileName
      
      switch (exampleType) {
        case 'brazil-states':
          exampleData = this.getBrazilStatesExample()
          fileName = 'estados-brasil-exemplo.geojson'
          break
        case 'sample-points':
          exampleData = this.getSamplePointsExample()
          fileName = 'pontos-exemplo.geojson'
          break
        default:
          this.errorMessage = 'Exemplo não encontrado'
          return
      }
      
      this.$emit('geojson-uploaded', exampleData, fileName)
      this.successMessage = `Exemplo "${fileName}" carregado!`
    },

    readFileAsText(file) {
      return new Promise((resolve, reject) => {
        const reader = new FileReader()
        reader.onload = e => resolve(e.target.result)
        reader.onerror = e => reject(new Error('Erro ao ler arquivo'))
        reader.readAsText(file)
      })
    },

    isValidGeoJSON(data) {
      // Validação básica de estrutura GeoJSON
      if (!data || typeof data !== 'object') return false
      
      if (data.type === 'FeatureCollection') {
        return Array.isArray(data.features)
      }
      
      if (data.type === 'Feature') {
        return data.geometry && data.geometry.type && data.geometry.coordinates
      }
      
      // Geometria direta
      const validGeometryTypes = [
        'Point', 'LineString', 'Polygon', 
        'MultiPoint', 'MultiLineString', 'MultiPolygon', 
        'GeometryCollection'
      ]
      
      return validGeometryTypes.includes(data.type) && data.coordinates
    },

    getBrazilStatesExample() {
      // Exemplo simplificado de alguns estados do Brasil
      return {
        type: 'FeatureCollection',
        features: [
          {
            type: 'Feature',
            properties: {
              name: 'São Paulo',
              region: 'Sudeste',
              population: 46649132
            },
            geometry: {
              type: 'Point',
              coordinates: [-46.6333, -23.5505]
            }
          },
          {
            type: 'Feature',
            properties: {
              name: 'Rio de Janeiro',
              region: 'Sudeste',
              population: 17366189
            },
            geometry: {
              type: 'Point',
              coordinates: [-43.1729, -22.9068]
            }
          },
          {
            type: 'Feature',
            properties: {
              name: 'Minas Gerais',
              region: 'Sudeste',
              population: 21411923
            },
            geometry: {
              type: 'Point',
              coordinates: [-43.9378, -19.9167]
            }
          }
        ]
      }
    },

    getSamplePointsExample() {
      // Pontos de interesse no Brasil
      return {
        type: 'FeatureCollection',
        features: [
          {
            type: 'Feature',
            properties: {
              name: 'Cristo Redentor',
              city: 'Rio de Janeiro',
              type: 'Monument'
            },
            geometry: {
              type: 'Point',
              coordinates: [-43.2105, -22.9519]
            }
          },
          {
            type: 'Feature',
            properties: {
              name: 'Cataratas do Iguaçu',
              city: 'Foz do Iguaçu',
              type: 'Natural'
            },
            geometry: {
              type: 'Point',
              coordinates: [-54.4367, -25.6953]
            }
          },
          {
            type: 'Feature',
            properties: {
              name: 'Teatro Amazonas',
              city: 'Manaus',
              type: 'Cultural'
            },
            geometry: {
              type: 'Point',
              coordinates: [-60.0231, -3.1301]
            }
          }
        ]
      }
    },

    clearMessages() {
      this.errorMessage = ''
      this.successMessage = ''
    }
  }
}
</script>

<style scoped>
.upload-area {
  border: 2px dashed #ddd;
  border-radius: 8px;
  padding: 2rem 1rem;
  text-align: center;
  cursor: pointer;
  transition: all 0.3s ease;
  margin-bottom: 1rem;
}

.upload-area:hover {
  border-color: #007bff;
  background-color: #f8f9fa;
}

.upload-area.dragover {
  border-color: #007bff;
  background-color: #e3f2fd;
  transform: scale(1.02);
}

.upload-content {
  pointer-events: none;
}

.upload-icon {
  font-size: 2rem;
  margin-bottom: 0.5rem;
}

.upload-area p {
  margin: 0.5rem 0;
  color: #666;
  font-weight: 500;
}

.upload-area small {
  color: #999;
  font-size: 0.8rem;
}

.loading-message {
  text-align: center;
  color: #007bff;
  font-style: italic;
  margin: 0.5rem 0;
}

.url-input-section {
  margin-top: 1.5rem;
  padding-top: 1rem;
  border-top: 1px solid #eee;
}

.url-input-section h4 {
  font-size: 0.9rem;
  color: #666;
  margin-bottom: 0.5rem;
}

.url-input-group {
  display: flex;
  gap: 0.5rem;
}

.url-input {
  flex: 1;
  padding: 0.5rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 0.85rem;
}

.examples-section {
  margin-top: 1.5rem;
  padding-top: 1rem;
  border-top: 1px solid #eee;
}

.examples-section h4 {
  font-size: 0.9rem;
  color: #666;
  margin-bottom: 0.5rem;
}

.btn-sm {
  padding: 0.3rem 0.6rem;
  font-size: 0.8rem;
  margin: 0.2rem;
}
</style>