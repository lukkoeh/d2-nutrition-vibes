<template>
  <div class="ml-panel">
    <ErrorBoundary @error="handleError">
      <div class="panel-header">
        <h2 class="panel-title">ML Prognosen</h2>
        <p class="panel-description">
          Machine Learning Vorhersagen für landwirtschaftliche Produktion
        </p>
      </div>

      <div class="panel-controls">
        <div class="filters-grid">
          <div class="filter-group">
            <label>Prognosentyp</label>
            <SearchableSelect
              v-model="selectedForecastType"
              :options="forecastTypeOptions"
              placeholder="Prognosentyp auswählen..."
              @update:modelValue="loadAvailableForecasts"
            />
          </div>
          
          <div class="filter-group">
            <label>Prognose</label>
            <SearchableSelect
              v-model="selectedForecast"
              :options="forecastOptions"
              placeholder="Prognose auswählen..."
              :disabled="availableForecasts.length === 0"
            />
          </div>
          
          <div class="filter-group">
            <label>Modell</label>
            <SearchableSelect
              v-model="selectedModel"
              :options="modelOptions"
              placeholder="Modell auswählen..."
            />
          </div>

          <div class="filter-group">
            <label class="invisible">Aktion</label>
            <BaseButton @click="loadPredictions" :disabled="isLoading || !selectedForecast" class="w-full">
              <LoadingSpinner v-if="isLoading" size="sm" />
              {{ isLoading ? 'Lädt...' : 'Prognosen laden' }}
            </BaseButton>
          </div>
        </div>
      </div>

      <div class="panel-content">
        <div v-if="hasError" class="error-container">
          <ErrorDisplay
            :error="error"
            title="Fehler beim Laden der ML-Prognosen"
            :showRetry="true"
            @retry="loadPredictions"
          />
        </div>

        <div v-else-if="isLoading" class="loading-container">
          <LoadingSpinner size="lg" />
          <p class="loading-text">Lade ML-Prognosen...</p>
        </div>

        <div v-else-if="predictions.length > 0" class="predictions-content">
          <!-- Forecast Info -->
          <div v-if="forecastData" class="forecast-info">
            <h3 class="info-title">{{ getFormattedForecastTitle() }}</h3>
            <div class="info-grid">
              <div class="info-item">
                <span class="info-label">Einheit:</span>
                <span class="info-value">{{ forecastData.unit || '1000 t' }}</span>
              </div>
              <div class="info-item">
                <span class="info-label">Prognosezeitraum:</span>
                <span class="info-value">2023 - 2035</span>
              </div>
              <div class="info-item">
                <span class="info-label">Modelltyp:</span>
                <span class="info-value">{{ getModelLabel(selectedModel) }}</span>
              </div>
              <div class="info-item">
                <span class="info-label">Konfidenzintervall:</span>
                <span class="info-value">95%</span>
              </div>
            </div>
          </div>

          <!-- Model Performance -->
          <div class="model-stats">
            <h3 class="stats-title">Modell-Performance</h3>
            <div v-if="modelStats.r2 && parseFloat(modelStats.r2) < 0" class="model-warning">
              <p class="warning-title">⚠️ Warnung: Schlechte Modellperformance</p>
              <p class="warning-text">
                Das lineare Modell ist für diese Daten ungeeignet. Bitte wechseln Sie zum polynomialen oder Ensemble-Modell für bessere Vorhersagen.
              </p>
            </div>
            <!-- Ensemble Info -->
            <div v-if="selectedModel === 'ensemble'" class="ensemble-info">
              <p class="text-sm text-blue-600 dark:text-blue-400 mb-3">
                ℹ️ Ensemble-Modell kombiniert Linear- und Polynomialregressionen für robustere Vorhersagen.
              </p>
            </div>
            
            <div class="stats-grid">
              <div class="stat-card">
                <div class="stat-label">R² Score</div>
                <div class="stat-value" :class="getR2Class(modelStats.r2)">{{ modelStats.r2 }}</div>
                <div v-if="selectedModel === 'ensemble'" class="stat-help text-xs text-gray-500 mt-1">
                  Durchschnitt der Einzelmodelle
                </div>
              </div>
              <div class="stat-card">
                <div class="stat-label">RMSE</div>
                <div class="stat-value">{{ modelStats.rmse }}</div>
                <div v-if="selectedModel === 'ensemble'" class="stat-help text-xs text-gray-500 mt-1">
                  Gemittelte Wurzel der mittleren Quadratfehler
                </div>
              </div>
              <div class="stat-card">
                <div class="stat-label">MAE</div>
                <div class="stat-value">{{ modelStats.mae }}</div>
                <div v-if="selectedModel === 'ensemble'" class="stat-help text-xs text-gray-500 mt-1">
                  Gemittelter mittlerer absoluter Fehler
                </div>
              </div>
              <div class="stat-card">
                <div class="stat-label">{{ selectedModel === 'ensemble' ? 'Übereinstimmung' : 'Genauigkeit' }}</div>
                <div class="stat-value text-green-600">{{ modelStats.accuracy }}%</div>
                <div v-if="selectedModel === 'ensemble'" class="stat-help text-xs text-gray-500 mt-1">
                  Modellübereinstimmung zwischen Linear- und Polynomialregression
                </div>
              </div>
            </div>
          </div>

          <!-- Predictions Chart -->
          <div class="chart-container">
            <MLChart
              :data="chartData"
              :config="chartConfig"
              @prediction-select="handlePredictionSelect"
              @confidence-toggle="handleConfidenceToggle"
            />
          </div>

          <!-- Insights Section -->
          <div v-if="forecastData" class="insights-section">
            <h3 class="insights-title">Prognose-Einblicke</h3>
            <div class="insights-grid">
              <div class="insight-card">
                <div class="insight-icon">📈</div>
                <div class="insight-content">
                  <h4>Wachstumstrend</h4>
                  <p>{{ getGrowthInsight() }}</p>
                </div>
              </div>
              <div class="insight-card">
                <div class="insight-icon">🎯</div>
                <div class="insight-content">
                  <h4>Prognosegenauigkeit</h4>
                  <p>{{ getAccuracyInsight() }}</p>
                </div>
              </div>
              <div class="insight-card">
                <div class="insight-icon">⚠️</div>
                <div class="insight-content">
                  <h4>Unsicherheitsfaktoren</h4>
                  <p>{{ getUncertaintyInsight() }}</p>
                </div>
              </div>
              <div class="insight-card">
                <div class="insight-icon">🌍</div>
                <div class="insight-content">
                  <h4>Globale Bedeutung</h4>
                  <p>{{ getGlobalInsight() }}</p>
                </div>
              </div>
            </div>
          </div>

          <!-- Predictions Table -->
          <div class="predictions-table">
            <h3 class="table-title">Detaillierte Prognosen</h3>
            <div class="table-container">
              <table class="predictions-table-element">
                <thead>
                  <tr>
                    <th>Jahr</th>
                    <th>Prognose</th>
                    <th>Konfidenzintervall</th>
                    <th>Trend</th>
                    <th>Zuverlässigkeit</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="prediction in predictions" :key="prediction.year">
                    <td>{{ prediction.year }}</td>
                    <td>{{ formatValue(prediction.predicted_value) }}</td>
                    <td>
                      {{ formatValue(prediction.confidence_lower) }} - 
                      {{ formatValue(prediction.confidence_upper) }}
                    </td>
                    <td>
                      <span :class="getTrendClass(prediction.trend)">
                        {{ formatTrend(prediction.trend) }}
                      </span>
                    </td>
                    <td>
                      <div class="reliability-bar">
                        <div 
                          class="reliability-fill"
                          :style="{ width: `${prediction.reliability}%` }"
                          :class="getReliabilityClass(prediction.reliability)"
                        ></div>
                        <span class="reliability-text">{{ prediction.reliability }}%</span>
                      </div>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>
        </div>

        <div v-else class="empty-state">
          <div class="empty-icon">🤖</div>
          <h3>Keine Prognosen geladen</h3>
          <p>Wählen Sie einen Prognosentyp und eine spezifische Prognose aus, dann klicken Sie auf "Prognosen laden".</p>
        </div>
      </div>
    </ErrorBoundary>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue'
import { useDataStore } from '@/stores/useDataStore'
import { useErrorHandling } from '@/composables/useErrorHandling'
import { getMLGermanName, productMappings } from '@/utils/productMappings'
import ErrorBoundary from '@/components/ui/ErrorBoundary.vue'
import ErrorDisplay from '@/components/ui/ErrorDisplay.vue'
import LoadingSpinner from '@/components/ui/LoadingSpinner.vue'
import BaseButton from '@/components/ui/BaseButton.vue'
import SearchableSelect from '@/components/ui/SearchableSelect.vue'
import MLChart from '@/components/visualizations/MLChart.vue'

// Store and composables
const dataStore = useDataStore()
const { handleError: handleErrorUtil, wrapAsync } = useErrorHandling()

// Reactive state
const selectedForecastType = ref('global')
const selectedForecast = ref('') // Will be set after loading forecasts
const selectedModel = ref('linear') // Will be updated based on model performance
const predictions = ref([])
const modelStats = ref({})
const isLoading = ref(false)
const error = ref(null)
const forecastData = ref(null)
const availableForecasts = ref([])

// Computed properties
const hasError = computed(() => error.value !== null)

const forecastTypeOptions = computed(() => [
  { value: 'global', label: 'Globale Prognosen' },
  { value: 'regional', label: 'Regionale Prognosen' },
  { value: 'country', label: 'Länder-spezifische Prognosen' }
])

const forecastOptions = computed(() => {
  console.log('Available forecasts:', availableForecasts.value)
  return availableForecasts.value.map(forecast => {
    const value = forecast.file.replace('.json', '')
    let label = forecast.title || forecast.scenario || forecast.file
    
    // Extract product name from the filename and get German translation
    if (typeof label === 'string') {
      // Extract product part from filename (remove prefix and suffix)
      // Handle patterns like: global_maize_and_products_forecast, world_cereals___excluding_beer_forecast, etc.
      const cleanValue = value.replace(/_forecast$/, '')
      let productName = ''
      let regionPrefix = ''
      
      if (cleanValue.startsWith('global_')) {
        productName = cleanValue.replace('global_', '')
        regionPrefix = 'Globale'
      } else if (cleanValue.startsWith('world_')) {
        productName = cleanValue.replace('world_', '')
        regionPrefix = 'Welt'
      } else {
        // Regional or country forecasts - find the last part that matches a product
        const parts = cleanValue.split('_')
        // Try to find product pattern from the end
        for (let i = parts.length - 1; i >= 1; i--) {
          const potentialProduct = parts.slice(i).join('_')
          if (productMappings[potentialProduct] || potentialProduct.includes('and') || potentialProduct.includes('&')) {
            productName = potentialProduct
            regionPrefix = parts.slice(0, i).join('_').replace(/_/g, ' ')
              .split(' ')
              .map(word => word.charAt(0).toUpperCase() + word.slice(1))
              .join(' ')
            break
          }
        }
        
        // Fallback: assume last part is product
        if (!productName && parts.length > 1) {
          productName = parts[parts.length - 1]
          regionPrefix = parts.slice(0, -1).join('_').replace(/_/g, ' ')
            .split(' ')
            .map(word => word.charAt(0).toUpperCase() + word.slice(1))
            .join(' ')
        }
      }
      
      if (productName) {
        const germanProductName = getMLGermanName(productName)
        label = `${regionPrefix} ${germanProductName} Prognose`
      }
    }
    
    return {
      value,
      label
    }
  })
})

const modelOptions = computed(() => [
  { value: 'linear', label: 'Lineare Regression' },
  { value: 'polynomial', label: 'Polynomiale Regression' },
  { value: 'ensemble', label: 'Ensemble (Linear + Polynomial)' }
])

const chartConfig = computed(() => ({
  width: '100%',
  height: 500,
  margin: { top: 20, right: 30, bottom: 40, left: 80 },
  showConfidenceInterval: true,
  showHistorical: true,
  showTrend: true,
  interactive: true,
  animated: true
}))

// Methods
const loadPredictions = wrapAsync(async () => {
  error.value = null
  isLoading.value = true
  
  try {
    console.log('Loading forecast:', selectedForecast.value)
    // Load ML forecast data
    const data = await dataStore.loadMLForecast(selectedForecast.value)
    console.log('ML Forecast data loaded:', data)
    
    if (data) {
      forecastData.value = data
      
      // Log model performance for user information (no automatic switching)
      if (data.model_performance) {
        const linearR2 = data.model_performance.linear?.r2_score || -999
        const polyR2 = data.model_performance.polynomial?.r2_score || -999
        
        console.log('📊 MLPanel: Model performance:', {
          linear: linearR2,
          polynomial: polyR2,
          currentModel: selectedModel.value
        })
      }
      
      // Extract predictions based on selected model
      let forecastArray = []
      if (selectedModel.value === 'ensemble' && data.ensemble_forecasts) {
        console.log('Using ensemble forecasts')
        forecastArray = data.ensemble_forecasts
      } else if (data.forecasts && data.forecasts[selectedModel.value]) {
        console.log(`Using ${selectedModel.value} forecasts`)
        forecastArray = data.forecasts[selectedModel.value]
      } else {
        console.warn('No forecast data found for model:', selectedModel.value)
        console.log('Available forecast keys:', data.forecasts ? Object.keys(data.forecasts) : 'No forecasts object')
      }
      
      // Combine with historical data if available
      const historicalData = data.historical_data || []
      const lastHistoricalYear = historicalData.length > 0 
        ? historicalData[historicalData.length - 1].year 
        : new Date().getFullYear() - 1
      
      // Transform forecast data
      predictions.value = forecastArray.map(pred => {
        const isEnsemble = selectedModel.value === 'ensemble'
        const predictedValue = isEnsemble ? pred.ensemble_mean : pred.value
        
        console.log(`🔍 MLPanel: Processing prediction for ${pred.year}:`, {
          isEnsemble,
          originalValue: pred.value,
          ensembleMean: pred.ensemble_mean,
          predictedValue,
          confidenceLower: pred.confidence_lower,
          confidenceUpper: pred.confidence_upper
        })
        
        return {
          year: pred.year,
          predicted_value: predictedValue,
          confidence_lower: isEnsemble ? pred.model_min : pred.confidence_lower,
          confidence_upper: isEnsemble ? pred.model_max : pred.confidence_upper,
          trend: calculateTrend(pred, lastHistoricalYear),
          reliability: isEnsemble 
            ? (pred.model_agreement * 100) 
            : (100 - pred.uncertainty_percent),
          uncertainty_level: pred.uncertainty_level || getUncertaintyLevel(pred.uncertainty_percent),
          model: selectedModel.value
        }
      })
      
      // Extract model performance stats
      if (data.model_performance && data.model_performance[selectedModel.value]) {
        const perf = data.model_performance[selectedModel.value]
        modelStats.value = {
          accuracy: ((perf.r2_score || 0) * 100).toFixed(1),
          rmse: Math.sqrt(perf.mse || 0).toFixed(0),
          mae: (perf.mae || 0).toFixed(0),
          r2: (perf.r2_score || 0).toFixed(3)
        }
      } else if (selectedModel.value === 'ensemble' && data.model_performance) {
        // For ensemble: calculate average performance from individual models
        const models = Object.values(data.model_performance)
        if (models.length > 0) {
          const avgR2 = models.reduce((sum, m) => sum + (m.r2_score || 0), 0) / models.length
          const avgMSE = models.reduce((sum, m) => sum + (m.mse || 0), 0) / models.length
          const avgMAE = models.reduce((sum, m) => sum + (m.mae || 0), 0) / models.length
          
          modelStats.value = {
            accuracy: (avgR2 * 100).toFixed(1),
            rmse: Math.sqrt(avgMSE).toFixed(0),
            mae: avgMAE.toFixed(0),
            r2: avgR2.toFixed(3)
          }
        } else {
          // Fallback for ensemble without individual model performance
          const avgAgreement = predictions.value.reduce((sum, p) => sum + (p.reliability || 0), 0) / predictions.value.length
          modelStats.value = {
            accuracy: avgAgreement.toFixed(1),
            rmse: 'Ensemble',
            mae: 'Kombination',
            r2: 'Gemittelt'
          }
        }
      } else {
        // Unknown model or missing data
        modelStats.value = {
          accuracy: 'N/A',
          rmse: 'N/A',
          mae: 'N/A',
          r2: 'N/A'
        }
      }
    }
    
  } catch (err) {
    error.value = err
    predictions.value = []
    modelStats.value = {}
  } finally {
    isLoading.value = false
  }
}, {
  component: 'MLPanel',
  operation: 'loadPredictions'
})

// Helper functions
const calculateTrend = (prediction, baseYear) => {
  if (!prediction.years_ahead || prediction.years_ahead === 0) return 0
  // Simple trend calculation - could be enhanced
  return ((prediction.value || prediction.ensemble_mean || 0) / 100000 - 1) * 10
}

const getUncertaintyLevel = (uncertaintyPercent) => {
  if (uncertaintyPercent < 30) return 'low'
  if (uncertaintyPercent < 60) return 'medium'
  return 'high'
}

const handleError = (err) => {
  error.value = err
  handleErrorUtil(err, {
    component: 'MLPanel',
    action: 'component_error'
  })
}

const handlePredictionSelect = (prediction) => {
  console.log('Prediction selected:', prediction)
  // TODO: Show prediction details
}

const handleConfidenceToggle = (showConfidence) => {
  console.log('Confidence toggle:', showConfidence)
  // TODO: Update chart visibility
}

// Helper methods
const getFormattedForecastTitle = () => {
  if (!forecastData.value || !selectedForecast.value) {
    return 'Produktionsprognose'
  }
  
  // Extract product name from selected forecast
  const productMatch = selectedForecast.value.match(/(?:global_|world_|[a-z_]+_)?([a-z_&]+)(?:_forecast)?$/i)
  if (productMatch) {
    const productName = productMatch[1]
    const germanProductName = getMLGermanName(productName)
    
    if (selectedForecast.value.startsWith('global_')) {
      return `Globale ${germanProductName} Produktionsprognose`
    } else if (selectedForecast.value.startsWith('world_')) {
      return `Welt ${germanProductName} Produktionsprognose`
    } else {
      // Regional or country forecasts
      const regionPart = selectedForecast.value.replace(`_${productName}_forecast`, '').replace(`_${productName}`, '').replace(/_/g, ' ')
      const formattedRegion = regionPart
        .split(' ')
        .map(word => word.charAt(0).toUpperCase() + word.slice(1))
        .join(' ')
      return `${formattedRegion} ${germanProductName} Produktionsprognose`
    }
  }
  
  return forecastData.value.title || 'Produktionsprognose'
}

const formatValue = (value) => {
  return new Intl.NumberFormat('de-DE', {
    maximumFractionDigits: 0
  }).format(value)
}

const formatTrend = (trend) => {
  const sign = trend > 0 ? '+' : ''
  return `${sign}${trend.toFixed(1)}%`
}

const getTrendClass = (trend) => {
  if (trend > 0) return 'text-green-600'
  if (trend < 0) return 'text-red-600'
  return 'text-gray-600'
}

const getReliabilityClass = (reliability) => {
  if (reliability >= 90) return 'bg-green-500'
  if (reliability >= 75) return 'bg-yellow-500'
  return 'bg-red-500'
}

const getModelLabel = (model) => {
  const labels = {
    linear: 'Lineare Regression',
    polynomial: 'Polynomiale Regression',
    ensemble: 'Ensemble-Modell'
  }
  return labels[model] || model
}

const getStatsExplanation = (model) => {
  if (model === 'ensemble') {
    return {
      r2: 'Durchschnittlicher R² Score der Einzelmodelle',
      rmse: 'Durchschnittliche Wurzel der mittleren Quadratfehler',
      mae: 'Durchschnittlicher mittlerer absoluter Fehler',
      accuracy: 'Durchschnittliche Modellübereinstimmung (%)'
    }
  } else {
    return {
      r2: 'Bestimmtheitsmaß (R²) - Erklärt Varianz',
      rmse: 'Wurzel der mittleren Quadratfehler',
      mae: 'Mittlerer absoluter Fehler',
      accuracy: 'Modellgenauigkeit basierend auf R²'
    }
  }
}

const getR2Class = (r2) => {
  const value = parseFloat(r2)
  if (isNaN(value)) return ''
  if (value < 0) return 'text-red-800 font-bold' // Negative R² is very bad
  if (value >= 0.8) return 'text-green-600'
  if (value >= 0.6) return 'text-yellow-600'
  if (value >= 0.3) return 'text-orange-600'
  return 'text-red-600'
}

// Insight functions
const getGrowthInsight = () => {
  if (!predictions.value.length || !forecastData.value) return 'Keine Daten verfügbar'
  
  const firstPred = predictions.value[0].predicted_value
  const lastPred = predictions.value[predictions.value.length - 1].predicted_value
  const growthPercent = ((lastPred - firstPred) / firstPred * 100).toFixed(1)
  
  if (growthPercent > 0) {
    return `Die Produktion wird voraussichtlich um ${growthPercent}% bis 2035 steigen.`
  } else {
    return `Die Produktion wird voraussichtlich um ${Math.abs(growthPercent)}% bis 2035 sinken.`
  }
}

const getAccuracyInsight = () => {
  if (!modelStats.value.r2) return 'Keine Genauigkeitsdaten verfügbar'
  
  const r2 = parseFloat(modelStats.value.r2)
  if (r2 < 0) {
    return `WARNUNG: Das ${selectedModel.value === 'linear' ? 'lineare' : 'polynomiale'} Modell ist ungeeignet für diese Daten (R² = ${r2.toFixed(3)}). Die Vorhersagen sind sehr unzuverlässig. Versuchen Sie ein anderes Modell.`
  } else if (r2 >= 0.8) {
    return 'Das Modell zeigt eine sehr hohe Prognosegenauigkeit mit starker Korrelation zu historischen Daten.'
  } else if (r2 >= 0.6) {
    return 'Das Modell zeigt eine gute Prognosegenauigkeit mit moderater Korrelation zu historischen Daten.'
  } else if (r2 >= 0.3) {
    return 'Das Modell zeigt moderate Genauigkeit. Die Prognosen sollten als grobe Schätzungen betrachtet werden.'
  } else {
    return 'Das Modell zeigt begrenzte Genauigkeit. Die Prognosen sollten mit großer Vorsicht interpretiert werden.'
  }
}

const getUncertaintyInsight = () => {
  if (!predictions.value.length) return 'Keine Unsicherheitsdaten verfügbar'
  
  const avgUncertainty = predictions.value
    .filter(p => p.uncertainty_level)
    .reduce((sum, p) => {
      const level = p.uncertainty_level === 'low' ? 1 : p.uncertainty_level === 'medium' ? 2 : 3
      return sum + level
    }, 0) / predictions.value.length
  
  if (avgUncertainty < 1.5) {
    return 'Die Prognosen haben eine niedrige Unsicherheit und sind relativ zuverlässig.'
  } else if (avgUncertainty < 2.5) {
    return 'Die Prognosen haben eine moderate Unsicherheit. Langfristige Vorhersagen sollten regelmäßig aktualisiert werden.'
  } else {
    return 'Die Prognosen haben eine hohe Unsicherheit, besonders für spätere Jahre. Externe Faktoren können die Ergebnisse stark beeinflussen.'
  }
}

const getGlobalInsight = () => {
  if (!selectedForecast.value) return 'Keine globalen Kontextdaten verfügbar'
  
  // Extract product name and get insights based on product type
  const productMatch = selectedForecast.value.match(/(?:global_|world_|[a-z_]+_)?([a-z_&]+)(?:_forecast)?$/i)
  if (productMatch) {
    const productName = productMatch[1].toLowerCase()
    
    if (productName.includes('wheat')) {
      return 'Weizen ist eines der wichtigsten Grundnahrungsmittel weltweit und ernährt Milliarden von Menschen.'
    } else if (productName.includes('maize')) {
      return 'Mais ist sowohl für die menschliche Ernährung als auch als Tierfutter von entscheidender Bedeutung.'
    } else if (productName.includes('rice')) {
      return 'Reis ist das Hauptnahrungsmittel für mehr als die Hälfte der Weltbevölkerung.'
    } else if (productName.includes('milk')) {
      return 'Milchprodukte sind eine wichtige Proteinquelle und unterstützen die Ernährungssicherheit weltweit.'
    } else if (productName.includes('soyabeans') || productName.includes('soyabean')) {
      return 'Sojabohnen sind eine wichtige Protein- und Ölquelle und werden weltweit als Viehfutter verwendet.'
    } else if (productName.includes('sugar')) {
      return 'Zucker und Süßstoffe sind wichtige Energielieferanten und haben großen Einfluss auf die globale Ernährung.'
    } else if (productName.includes('vegetable') || productName.includes('fruits')) {
      return 'Obst und Gemüse sind essentiell für eine ausgewogene Ernährung und liefern wichtige Vitamine und Mineralstoffe.'
    } else if (productName.includes('meat') || productName.includes('bovine') || productName.includes('pigmeat') || productName.includes('poultry')) {
      return 'Fleischprodukte sind wichtige Protein- und Nährstoffquellen, deren Produktion erhebliche Umweltauswirkungen hat.'
    } else if (productName.includes('cassava')) {
      return 'Maniok ist ein wichtiges Grundnahrungsmittel in tropischen Regionen und eine wichtige Kohlenhydratquelle.'
    } else if (productName.includes('potatoes')) {
      return 'Kartoffeln sind weltweit ein wichtiges Grundnahrungsmittel und eine bedeutende Quelle für Kohlenhydrate.'
    } else if (productName.includes('fish') || productName.includes('seafood')) {
      return 'Fisch und Meeresfrüchte sind wichtige Protein- und Omega-3-Quellen für Milliarden von Menschen.'
    } else {
      return 'Diese Produktkategorie spielt eine wichtige Rolle für die globale Ernährungssicherheit und nachhaltige Landwirtschaft.'
    }
  }
  
  return 'Diese Produktkategorie spielt eine wichtige Rolle für die globale Ernährungssicherheit.'
}

// Combined data for chart
const chartData = computed(() => {
  if (!forecastData.value) return []
  
  // Filter out historical data with invalid values
  const historical = forecastData.value.historical_data?.map(d => ({
    year: d.year,
    value: d.value,
    predicted_value: d.value, // Add for consistency with chart expectations
    type: 'historical'
  })).filter(d => d.value != null && d.value > 0) || []
  
  // Filter out predictions with invalid values
  const validPredictions = predictions.value.filter(d => 
    d.predicted_value != null && d.predicted_value > 0
  ).map(d => ({
    ...d,
    value: d.predicted_value, // Add for consistency
    type: 'prediction'
  }))
  
  const combined = [...historical, ...validPredictions]
  console.log('🎯 MLPanel: Chart data:', { 
    historical: historical.length, 
    predictions: validPredictions.length,
    sample: combined.slice(0, 3)
  })
  return combined
})

// Watchers
watch([selectedForecast, selectedModel], () => {
  if (selectedForecast.value && selectedModel.value) {
    loadPredictions()
  }
}, { deep: true })

// Load available forecasts based on type
const loadAvailableForecasts = wrapAsync(async () => {
  console.log('🔍 MLPanel: Loading forecasts for type:', selectedForecastType.value)
  try {
    let index
    switch (selectedForecastType.value) {
      case 'global':
        index = await dataStore.loadMLGlobalIndex()
        break
      case 'regional':
        index = await dataStore.loadMLRegionalIndex()
        break
      case 'country':
        index = await dataStore.loadMLCountryIndex()
        break
    }
    
    console.log('📋 MLPanel: ML Index loaded:', index)
    
    // Handle both 'forecasts' and 'files' formats
    if (index && (index.forecasts || index.files)) {
      const forecasts = index.forecasts || index.files
      
      // Transform files array to expected format if needed
      if (Array.isArray(forecasts) && typeof forecasts[0] === 'string') {
        availableForecasts.value = forecasts.map(file => ({
          file: file,
          title: file
            .replace('.json', '')
            .replace(/_/g, ' ')
            .replace(/global /i, '')
            .replace(/forecast/i, '')
            .trim()
            .split(' ')
            .map(word => word.charAt(0).toUpperCase() + word.slice(1))
            .join(' ')
        }))
      } else {
        availableForecasts.value = forecasts
      }
      
      // Select first forecast if available
      if (availableForecasts.value.length > 0 && !selectedForecast.value) {
        selectedForecast.value = availableForecasts.value[0].file.replace('.json', '')
        console.log('✅ MLPanel: Auto-selected forecast:', selectedForecast.value)
      }
      console.log('📊 MLPanel: Available forecasts:', availableForecasts.value.length)
    } else {
      console.warn('⚠️ MLPanel: No forecasts found in index')
      availableForecasts.value = []
    }
  } catch (err) {
    console.error('❌ MLPanel: Error loading forecast index:', err)
    availableForecasts.value = []
  }
}, {
  component: 'MLPanel',
  operation: 'loadAvailableForecasts'
})

// Lifecycle
onMounted(async () => {
  try {
    // Load comprehensive index on mount
    const comprehensiveIndex = await dataStore.loadMLComprehensiveIndex()
    console.log('Comprehensive index loaded:', comprehensiveIndex)
  } catch (err) {
    console.error('Failed to load comprehensive index:', err)
  }
  
  await loadAvailableForecasts()
  if (selectedForecast.value) {
    await loadPredictions()
  }
})
</script>

<style scoped>
.ml-panel {
  @apply flex flex-col h-full bg-white dark:bg-gray-800 rounded-lg shadow-lg;
}

.panel-header {
  @apply p-6 border-b border-gray-200 dark:border-gray-700;
}

.panel-title {
  @apply text-2xl font-bold text-gray-900 dark:text-gray-100 mb-2;
}

.panel-description {
  @apply text-gray-600 dark:text-gray-300;
}

.panel-controls {
  @apply p-6 bg-gray-50 dark:bg-gray-700 border-b border-gray-200 dark:border-gray-600;
}

.filters-grid {
  @apply grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4;
}

.filter-group {
  @apply flex flex-col;
}

.filter-group label {
  @apply text-sm font-medium text-gray-700 dark:text-gray-300 mb-2;
}

.panel-content {
  @apply flex-1 p-6 overflow-auto;
}

.predictions-content {
  @apply space-y-8;
}

.forecast-info {
  @apply bg-blue-50 dark:bg-blue-900/30 rounded-lg p-6 mb-6;
}

.info-title {
  @apply text-xl font-bold text-gray-900 dark:text-gray-100 mb-4;
}

.info-grid {
  @apply grid grid-cols-2 md:grid-cols-4 gap-4;
}

.info-item {
  @apply flex flex-col;
}

.info-label {
  @apply text-sm text-gray-600 dark:text-gray-400 mb-1;
}

.info-value {
  @apply text-base font-semibold text-gray-900 dark:text-gray-100;
}

.model-stats {
  @apply mb-6;
}

.stats-title {
  @apply text-lg font-semibold text-gray-900 dark:text-gray-100 mb-4;
}

.stats-grid {
  @apply grid grid-cols-2 md:grid-cols-4 gap-4;
}

.stat-card {
  @apply bg-gray-50 dark:bg-gray-700 rounded-lg p-4 text-center;
}

.stat-label {
  @apply text-sm text-gray-600 dark:text-gray-400 mb-1;
}

.stat-value {
  @apply text-lg font-bold text-gray-900 dark:text-gray-100;
}

.chart-container {
  @apply w-full mb-6;
  min-height: 500px;
}

.insights-section {
  @apply mb-8;
}

.insights-title {
  @apply text-lg font-semibold text-gray-900 dark:text-gray-100 mb-4;
}

.insights-grid {
  @apply grid grid-cols-1 md:grid-cols-2 gap-4;
}

.insight-card {
  @apply flex items-start space-x-4 p-4 bg-gray-50 dark:bg-gray-700 rounded-lg;
}

.insight-icon {
  @apply text-2xl;
}

.insight-content h4 {
  @apply font-semibold text-gray-900 dark:text-gray-100 mb-1;
}

.insight-content p {
  @apply text-sm text-gray-600 dark:text-gray-300;
}

.predictions-table {
  @apply space-y-4;
}

.table-title {
  @apply text-lg font-semibold text-gray-900 dark:text-gray-100;
}

.table-container {
  @apply overflow-x-auto;
}

.predictions-table-element {
  @apply w-full border-collapse bg-white dark:bg-gray-800 rounded-lg overflow-hidden shadow;
}

.predictions-table-element th {
  @apply bg-gray-50 dark:bg-gray-700 px-4 py-3 text-left text-sm font-medium text-gray-700 dark:text-gray-300 border-b border-gray-200 dark:border-gray-600;
}

.predictions-table-element td {
  @apply px-4 py-3 text-sm text-gray-900 dark:text-gray-100 border-b border-gray-200 dark:border-gray-600;
}

.reliability-bar {
  @apply relative w-full h-4 bg-gray-200 dark:bg-gray-600 rounded-full overflow-hidden;
}

.reliability-fill {
  @apply absolute top-0 left-0 h-full transition-all duration-300;
}

.reliability-text {
  @apply absolute inset-0 flex items-center justify-center text-xs font-medium text-white;
}

.error-container,
.loading-container,
.empty-state {
  @apply flex flex-col items-center justify-center h-64;
}

.loading-text {
  @apply mt-4 text-gray-600 dark:text-gray-400;
}

.empty-icon {
  @apply text-6xl mb-4;
}

.empty-state h3 {
  @apply text-xl font-semibold text-gray-700 dark:text-gray-300 mb-2;
}

.empty-state p {
  @apply text-gray-500 dark:text-gray-400;
}

/* Warning box for poor model performance */
.model-warning {
  @apply mb-4 p-4 bg-red-100 dark:bg-red-900/30 border border-red-400 dark:border-red-600 rounded-lg;
}

.model-warning p.warning-title {
  @apply text-red-700 dark:text-red-300 font-semibold;
}

.model-warning p.warning-text {
  @apply text-red-600 dark:text-red-400 text-sm mt-1;
}
</style>