<template>
  <div class="timeseries-panel">
    <ErrorBoundary @error="handleError">
      <div class="panel-header">
        <h2 class="panel-title">Zeitreihen-Analyse</h2>
        <p class="panel-description">
          Zeitliche Entwicklung von Produktionsdaten und Trends
        </p>
        
        <div class="simulation-info">
          <div class="info-section">
            <h3 class="info-title">Was wird berechnet?</h3>
            <p class="info-text">
              Diese Analyse zeigt die <strong>historische Entwicklung</strong> verschiedener landwirtschaftlicher Metriken über Zeit. Die Daten stammen aus der FAO-Datenbank und umfassen Produktions-, Import-, Export- und Versorgungsdaten für verschiedene Länder und Regionen von 1961 bis heute.
            </p>
          </div>
          
          <div class="info-section">
            <h3 class="info-title">Wie benutzt man es?</h3>
            <div class="info-steps">
              <div class="step">
                <span class="step-number">1</span>
                <span class="step-text"><strong>Produkte wählen:</strong> Wählen Sie bis zu 3 landwirtschaftliche Produkte aus der Liste</span>
              </div>
              <div class="step">
                <span class="step-number">2</span>
                <span class="step-text"><strong>Länder auswählen:</strong> Wählen Sie bis zu 5 Länder oder lassen Sie das Feld leer für globale Daten</span>
              </div>
              <div class="step">
                <span class="step-number">3</span>
                <span class="step-text"><strong>Metriken festlegen:</strong> Wählen Sie bis zu 3 Metriken wie Produktion, Import, Export oder Kalorienversorgung</span>
              </div>
            </div>
          </div>
          
          <div class="info-section">
            <h3 class="info-title">Berechnungsgrundlage</h3>
            <p class="info-text">
              Die Zeitreihendaten basieren auf offiziellen <strong>FAO-Statistiken</strong> und werden in verschiedenen Einheiten dargestellt (Tonnen, Kalorien/Person/Tag). Fehlende Daten werden als solche markiert. Globale Werte werden durch Summierung aller verfügbaren Länderdaten berechnet.
            </p>
          </div>
        </div>
      </div>

      <div class="panel-controls">
        <div class="filters-grid">
          <div class="filter-group">
            <MultiSelect
              v-model="selectedProducts"
              :options="productOptions"
              placeholder="Produkte auswählen..."
              label="Produkte"
              size="md"
              :max-items="3"
            />
          </div>
          
          <div class="filter-group">
            <MultiSelect
              v-model="selectedCountries"
              :options="countryOptions"
              placeholder="Länder auswählen..."
              label="Länder"
              size="md"
              :max-items="5"
            />
          </div>
          
          <div class="filter-group" data-tour="metric-selector">
            <MultiSelect
              v-model="selectedMetrics"
              :options="metricOptions"
              placeholder="Metriken auswählen..."
              label="Metriken"
              size="md"
              :max-items="3"
            />
          </div>
        </div>
      </div>

      <div class="panel-content">
        <div v-if="hasError" class="error-container">
          <ErrorDisplay
            :error="error"
            title="Fehler beim Laden der Zeitreihendaten"
            :show-retry="true"
            @retry="ensureDataLoaded"
          />
        </div>

        <div v-else-if="chartData.length > 0 || missingDataCombinations.length > 0" class="content-wrapper">
          <div v-if="missingDataCombinations.length > 0" class="missing-data-warning">
            <div class="warning-header">
              <span class="warning-icon">⚠️</span>
              <span class="warning-title">Hinweis: Einige Datenkombinationen sind nicht verfügbar</span>
              <button 
                class="toggle-details-btn"
                @click="showMissingDetails = !showMissingDetails"
              >
                {{ showMissingDetails ? 'Details ausblenden' : 'Details anzeigen' }}
              </button>
            </div>
            <div v-if="showMissingDetails" class="missing-details">
              <div 
                v-for="(item, index) in missingDataCombinations" 
                :key="index"
                class="missing-item"
              >
                <span class="missing-label">{{ item.product }}</span>
                <span class="missing-separator">-</span>
                <span class="missing-label">{{ item.country }}</span>
                <span class="missing-separator">-</span>
                <span class="missing-label">{{ item.metric }}</span>
                <span class="missing-reason">({{ item.reason }})</span>
              </div>
            </div>
          </div>
          
          <div v-if="chartData.length > 0" class="chart-container" data-tour="timeseries-chart">
            <TimeseriesChart
              :width="800"
              :height="500"
              :selected-countries="selectedCountries"
              :selected-products="selectedProducts"
              :selected-metrics="selectedMetrics"
              :chart-data="chartData"
              @point-click="handlePointClick"
            />
          </div>
        </div>

        <div v-else class="empty-state">
          <div class="empty-icon">📈</div>
          <h3>Keine Daten verfügbar</h3>
          <p v-if="selectedProducts.length === 0">
            Wählen Sie mindestens ein Produkt aus, um Zeitreihendaten anzuzeigen.
          </p>
          <p v-else>
            Die ausgewählten Produkte haben keine verfügbaren Zeitreihendaten.
          </p>
        </div>
      </div>

      <div v-if="chartData.length > 0" class="panel-footer">
        <div class="stats-grid">
          <div class="stat-item">
            <span class="stat-label">Datenpunkte</span>
            <span class="stat-value">{{ chartData.length }}</span>
          </div>
          <div class="stat-item">
            <span class="stat-label">{{ selectedProducts.length > 1 ? 'Produkte' : 'Produkt' }}</span>
            <span class="stat-value">{{ selectedProducts.length }}</span>
          </div>
          <div class="stat-item">
            <span class="stat-label">{{ selectedCountries.length > 1 ? 'Länder' : 'Land' }}</span>
            <span class="stat-value">{{ selectedCountries.length > 0 ? selectedCountries.length : 'Global' }}</span>
          </div>
        </div>
      </div>
    </ErrorBoundary>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue'
import { useDataStore } from '@/stores/useDataStore'
import { useUIStore } from '@/stores/useUIStore'
import ErrorBoundary from '@/components/ui/ErrorBoundary.vue'
import ErrorDisplay from '@/components/ui/ErrorDisplay.vue'
import SearchableSelect from '@/components/ui/SearchableSelect.vue'
import MultiSelect from '@/components/ui/MultiSelect.vue'
import TimeseriesChart from '@/components/visualizations/TimeseriesChart.vue'
import { getAllProductOptions, getGermanName } from '@/utils/productMappings'
import urlService from '@/services/urlState.js'
import { useRoute } from 'vue-router'

// Store and composables
const dataStore = useDataStore()
const uiStore = useUIStore()
const route = useRoute()

// Reactive state - use the same defaults as dashboard
const chartData = ref([])
const error = ref(null)
const missingDataCombinations = ref([])
const showMissingDetails = ref(false)

// Computed properties
const hasError = computed(() => error.value !== null)

// Get available products from metadata
const productOptions = computed(() => {
  const individualItems = (dataStore.faoMetadata?.data_summary?.food_items || []).filter(p => p !== 'Animal Products')
  
  const allOption = { value: 'All', label: 'Alle Produkte' }
  
  if (individualItems.length > 0) {
    const productList = individualItems.map(product => ({
      value: product,
      label: getGermanName(product)
    })).sort((a, b) => a.label.localeCompare(b.label, 'de'))
    
    return [allOption, ...productList]
  } else {
    const productList = getAllProductOptions().filter(opt => opt.value !== 'Animal Products')
    return [allOption, ...productList]
  }
})

// Get available countries from timeseries data
const countryOptions = computed(() => {
  if (!dataStore.timeseriesData || selectedProducts.value.length === 0) return []
  
  // Get union of all countries that exist for any selected product
  const allCountries = new Set()
  
  selectedProducts.value.forEach(product => {
    const productData = dataStore.timeseriesData[product]
    if (productData) {
      Object.keys(productData).forEach(country => allCountries.add(country))
    }
  })
  
  const NON_COUNTRY_ENTITIES = [
    "World", "Africa", "Americas", "Asia", "Europe", "Oceania",
    "Northern America", "South America", "Central America", "Caribbean",
    "Northern Africa", "Eastern Africa", "Middle Africa", "Southern Africa", "Western Africa", 
    "Eastern Asia", "South-eastern Asia", "Southern Asia", "Western Asia", "Central Asia",
    "Eastern Europe", "Northern Europe", "Southern Europe", "Western Europe",
    "Australia and New Zealand", "Melanesia",
    "European Union (27)",
    "Small Island Developing States", "Least Developed Countries", 
    "Land Locked Developing Countries", "Low Income Food Deficit Countries",
    "Net Food Importing Developing Countries"
  ]
  
  const allOption = { value: 'All', label: 'Alle Länder' }
  
  const countryList = [...allCountries]
    .filter(country => 
      country !== 'All' && // Exclude the pre-computed "All" key
      !NON_COUNTRY_ENTITIES.includes(country) && 
      !country.toLowerCase().includes('total')
    )
    .map(country => ({
      value: country,
      label: country
    }))
    .sort((a, b) => a.label.localeCompare(b.label))
  
  return [allOption, ...countryList]
})

// Metric options
const metricOptions = [
  { value: 'production', label: 'Produktion' },
  { value: 'import_quantity', label: 'Import' },
  { value: 'export_quantity', label: 'Export' },
  { value: 'domestic_supply_quantity', label: 'Inlandsversorgung' },
  { value: 'food_supply_kcal', label: 'Kalorienversorgung' },
  { value: 'feed', label: 'Tierfutter' },
  { value: 'protein', label: 'Protein (1000 t)' },
  { value: 'protein_gpcd', label: 'Protein (g/Kopf/Tag)' },
  { value: 'fat', label: 'Fett (1000 t)' },
  { value: 'fat_gpcd', label: 'Fett (g/Kopf/Tag)' },
  { value: 'processing', label: 'Verarbeitung (1000 t)' },
  { value: 'feed_share', label: 'Tierfutteranteil (%)' }
]

// Helper zur Berechnung des Tierfutteranteils (%)
const computeFeedShare = (entry) => {
  const ds = entry.domestic_supply || 0
  const feed = entry.feed || 0
  return ds > 0 ? (feed / ds) * 100 : 0
}

// Helper zum Parsen von Query → Array
const toArr = (param) => {
  if (!param) return []
  return decodeURIComponent(String(param)).split(',').filter(Boolean)
}

// Initialwerte aus UrlStateService (Timeseries-spezifische Keys)
const selectedProducts = ref(urlService._tsState.tpr?.length ? [...urlService._tsState.tpr] : ['Wheat and products'])
const selectedCountries = ref(urlService._tsState.tcty?.length ? [...urlService._tsState.tcty] : [])
const selectedMetrics = ref(urlService._tsState.tmet?.length ? [...urlService._tsState.tmet] : ['production'])

// Update chart data when selections change
const updateChartData = () => {
  if (!dataStore.timeseriesData || selectedProducts.value.length === 0 || selectedMetrics.value.length === 0) {
    chartData.value = []
    missingDataCombinations.value = []
    return
  }

  const allData = []
  const missingCombinations = []

  // Create arrays of all products and countries to process (including "All" and specific items)
  const productsToProcess = selectedProducts.value.slice() // Copy the array
  const countriesToProcess = selectedCountries.value.length > 0 ? selectedCountries.value.slice() : ['Global'] // Default to Global if no countries selected

  // Iterate through each metric
  selectedMetrics.value.forEach(metric => {
    const metricKey = metric === 'production' ? 'production' :
                     metric === 'import_quantity' ? 'imports' :
                     metric === 'export_quantity' ? 'exports' :
                     metric === 'domestic_supply_quantity' ? 'domestic_supply' :
                     metric === 'feed' ? 'feed' :
                     metric === 'food_supply_kcal' ? 'food_supply_kcal' :
                     metric === 'protein' ? 'protein' :
                     metric === 'protein_gpcd' ? 'protein_gpcd' :
                     metric === 'fat' ? 'fat' :
                     metric === 'fat_gpcd' ? 'fat_gpcd' :
                     metric === 'processing' ? 'processing' :
                     metric === 'feed_share' ? 'feed_share' :
                     'production'

    const metricLabel = metricOptions.find(m => m.value === metric)?.label || metric

    // Process each product-country combination
    productsToProcess.forEach(product => {
      countriesToProcess.forEach(country => {
        let seriesData = []
        let seriesName = ''
        let hasData = false

        if (product === 'All' && country === 'All') {
          // All Products + All Countries = Global aggregation
          if (dataStore.timeseriesData['All'] && dataStore.timeseriesData['All']['All']) {
            seriesData = dataStore.timeseriesData['All']['All']
            seriesName = `Alle Länder - Alle Produkte - ${metricLabel}`
          }
        } else if (product === 'All' && country === 'Global') {
          // All Products + No specific countries = Global aggregation
          if (dataStore.timeseriesData['All'] && dataStore.timeseriesData['All']['All']) {
            seriesData = dataStore.timeseriesData['All']['All']
            seriesName = `Alle Produkte - ${metricLabel}`
          }
        } else if (product === 'All' && country !== 'All' && country !== 'Global') {
          // All Products + Specific Country
          if (dataStore.timeseriesData['All'] && dataStore.timeseriesData['All'][country]) {
            seriesData = dataStore.timeseriesData['All'][country]
            seriesName = `${country} - Alle Produkte - ${metricLabel}`
          }
        } else if (product !== 'All' && country === 'All') {
          // Specific Product + All Countries
          const productData = dataStore.timeseriesData[product]
          if (productData && productData['All']) {
            seriesData = productData['All']
            seriesName = `Alle Länder - ${getGermanName(product)} - ${metricLabel}`
          }
        } else if (product !== 'All' && country === 'Global') {
          // Specific Product + No specific countries = Manual global aggregation
          const productData = dataStore.timeseriesData[product]
          if (productData) {
            const yearlyTotals = new Map()
            const yearlyFeed = new Map()
            const yearlyDS = new Map()
            
            Object.entries(productData).forEach(([countryKey, countryData]) => {
              if (countryKey !== 'All') { // Exclude pre-computed "All" to avoid double counting
                countryData.forEach(yearData => {
                  if (metric === 'feed_share') {
                    const year = yearData.year
                    const feedVal = yearData.feed || 0
                    const dsVal = yearData.domestic_supply || 0
                    yearlyFeed.set(year, (yearlyFeed.get(year) || 0) + feedVal)
                    yearlyDS.set(year, (yearlyDS.get(year) || 0) + dsVal)
                  } else {
                    const value = yearData[metricKey] || 0
                    if (value > 0) {
                      const year = yearData.year
                      const currentTotal = yearlyTotals.get(year) || 0
                      yearlyTotals.set(year, currentTotal + value)
                    }
                  }
                })
              }
            })
            
            // Convert to series data format
            if (metric === 'feed_share') {
              seriesData = Array.from(yearlyDS.entries()).map(([year, dsVal]) => {
                const feedVal = yearlyFeed.get(year) || 0
                const perc = dsVal > 0 ? (feedVal / dsVal) * 100 : 0
                return {
                  year: year,
                  feed_share: perc,
                  unit: '%'
                }
              })
            } else {
              seriesData = Array.from(yearlyTotals.entries()).map(([year, value]) => ({
                year: year,
                [metricKey]: value,
                unit: '1000 t'
              }))
            }
            seriesName = `Global - ${getGermanName(product)} - ${metricLabel}`
          }
        } else if (product !== 'All' && country !== 'All' && country !== 'Global') {
          // Specific Product + Specific Country
          const productData = dataStore.timeseriesData[product]
          if (productData && productData[country]) {
            seriesData = productData[country]
            seriesName = `${country} - ${getGermanName(product)} - ${metricLabel}`
          }
        }

        // Process the series data if available
        if (seriesData && seriesData.length > 0) {
          seriesData.forEach(yearData => {
            const value = metric === 'feed_share' ? computeFeedShare(yearData) : (yearData[metricKey] || 0)
            if (value > 0) {
              hasData = true
              allData.push({
                year: yearData.year,
                value: value,
                country: country,
                product: product,
                metric: metric,
                metricLabel: metricLabel,
                unit: yearData.unit || '1000 t',
                series: seriesName
              })
            }
          })
        }

        // Track missing data combinations
        if (!hasData && seriesData !== null) {
          const productName = product === 'All' ? 'Alle Produkte' : getGermanName(product)
          const countryName = country === 'All' ? 'Alle Länder' : 
                             country === 'Global' ? 'Global' : country
          
          missingCombinations.push({
            product: productName,
            country: countryName,
            metric: metricLabel,
            reason: seriesData && seriesData.length > 0 ? 'Keine Werte > 0' : 'Keine Daten verfügbar'
          })
        }
      })
    })
  })
  
  chartData.value = allData.sort((a, b) => {
    if (a.series !== b.series) return a.series.localeCompare(b.series)
    return a.year - b.year
  })
  
  missingDataCombinations.value = missingCombinations
}

// Methods
const ensureDataLoaded = async () => {
  try {
    if (!dataStore.timeseriesData) {
      await dataStore.initializeApp()
    }
  } catch (err) {
    error.value = err
    console.error('TimeseriesPanel data loading error:', err)
  }
}

const handleError = (err) => {
  error.value = err
  console.error('TimeseriesPanel error:', err)
}

const handlePointClick = (point) => {
  console.log('Point clicked:', point)
}

// Watchers
watch([selectedProducts, selectedCountries, selectedMetrics], () => {
  updateChartData()
}, { immediate: true })

// --- Watcher: UI → URL (Timeseries Keys) ------------------------------
watch(selectedProducts, (val) => {
  urlService._tsState.tpr = [...val]
}, { deep: true })

watch(selectedCountries, (val) => {
  urlService._tsState.tcty = [...val]
}, { deep: true })

watch(selectedMetrics, (val) => {
  urlService._tsState.tmet = [...val]
}, { deep: true })

// --- Watcher: Route(Query) → UI (Timeseries Keys) ---------------------
watch(() => [route.query.tpr, route.query.tcty, route.query.tmet], () => {
  // Arrays aus Query neu setzen; UrlStateService kümmert sich beim Deserialisieren darum
  if (urlService._tsState.tpr && JSON.stringify(urlService._tsState.tpr) !== JSON.stringify(selectedProducts.value)) {
    selectedProducts.value = [...urlService._tsState.tpr]
  }
  if (urlService._tsState.tcty && JSON.stringify(urlService._tsState.tcty) !== JSON.stringify(selectedCountries.value)) {
    selectedCountries.value = [...urlService._tsState.tcty]
  }
  if (urlService._tsState.tmet && JSON.stringify(urlService._tsState.tmet) !== JSON.stringify(selectedMetrics.value)) {
    selectedMetrics.value = [...urlService._tsState.tmet]
  }
}, { immediate: true })

// Lifecycle
onMounted(async () => {
  await ensureDataLoaded()
  updateChartData()
})
</script>

<style scoped>
.timeseries-panel {
  @apply flex flex-col h-full bg-white dark:bg-gray-800 rounded-lg shadow-lg;
}

.panel-header {
  @apply p-6 border-b border-gray-200 dark:border-gray-700;
}

.panel-title {
  @apply text-2xl font-bold text-gray-900 dark:text-gray-100 mb-2;
}

.panel-description {
  @apply text-gray-600 dark:text-gray-400;
}

.panel-controls {
  @apply p-6 bg-gray-50 dark:bg-gray-900 border-b border-gray-200 dark:border-gray-700;
}

.filters-grid {
  @apply grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6 items-start;
}

.filter-group {
  @apply flex flex-col;
}

.panel-content {
  @apply flex-1 p-6 min-h-0;
}

.error-container,
.empty-state {
  @apply flex flex-col items-center justify-center h-full;
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

.chart-container {
  @apply w-full bg-white dark:bg-gray-800 rounded-lg border border-gray-200 dark:border-gray-700;
  min-height: 500px;
  height: auto; /* Allow container to grow with content */
  padding: 24px; /* Add padding around chart and legend */
}

.panel-footer {
  @apply p-6 bg-gray-50 dark:bg-gray-900 border-t border-gray-200 dark:border-gray-700;
}

.stats-grid {
  @apply grid grid-cols-3 gap-4;
}

.stat-item {
  @apply text-center;
}

.stat-label {
  @apply block text-sm text-gray-500 dark:text-gray-400 mb-1;
}

.stat-value {
  @apply block text-lg font-semibold text-gray-900 dark:text-gray-100;
}

.content-wrapper {
  @apply flex flex-col h-full gap-4;
}

.missing-data-warning {
  @apply bg-yellow-50 dark:bg-yellow-900/20 border border-yellow-200 dark:border-yellow-800 rounded-lg p-4;
}

.warning-header {
  @apply flex items-center gap-2;
}

.warning-icon {
  @apply text-yellow-600 dark:text-yellow-400 text-lg;
}

.warning-title {
  @apply text-sm font-medium text-yellow-800 dark:text-yellow-200 flex-1;
}

.toggle-details-btn {
  @apply text-xs text-yellow-700 dark:text-yellow-300 hover:text-yellow-800 dark:hover:text-yellow-200 underline;
}

.missing-details {
  @apply mt-3 space-y-1;
}

.missing-item {
  @apply text-xs text-yellow-700 dark:text-yellow-300 flex items-center gap-1;
}

.missing-label {
  @apply font-medium;
}

.missing-separator {
  @apply text-yellow-600 dark:text-yellow-400;
}

.missing-reason {
  @apply text-yellow-600 dark:text-yellow-400 ml-1;
}

/* Information Cards Styles */
.simulation-info {
  @apply grid grid-cols-1 lg:grid-cols-3 gap-6 mt-6;
}

.info-section {
  @apply bg-gray-50 dark:bg-gray-900 rounded-lg p-4;
}

.info-title {
  @apply text-lg font-semibold text-gray-900 dark:text-gray-100 mb-3;
}

.info-text {
  @apply text-sm text-gray-600 dark:text-gray-400 leading-relaxed;
}

.info-steps {
  @apply space-y-3;
}

.step {
  @apply flex items-start space-x-3;
}

.step-number {
  @apply flex-shrink-0 w-6 h-6 bg-blue-500 dark:bg-blue-600 text-white text-xs font-bold rounded-full flex items-center justify-center;
}

.step-text {
  @apply text-sm text-gray-600 dark:text-gray-400 leading-relaxed;
}
</style>