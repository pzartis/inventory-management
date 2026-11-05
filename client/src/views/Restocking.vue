<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div class="card budget-card">
      <div class="budget-header">
        <label class="stat-label" for="budget-slider">{{ t('restocking.budget') }}</label>
        <div class="budget-value">{{ formatMoney(budget) }}</div>
      </div>
      <input
        id="budget-slider"
        v-model.number="budget"
        class="budget-slider"
        type="range"
        min="0"
        max="100000"
        step="1000"
        :aria-label="t('restocking.budgetSlider')"
      />
      <div class="budget-scale">
        <span>{{ formatMoney(0) }}</span>
        <span>{{ formatMoney(100000) }}</span>
      </div>
    </div>

    <div class="stats-grid">
      <div class="stat-card info">
        <div class="stat-label">{{ t('restocking.budget') }}</div>
        <div class="stat-value">{{ formatMoney(budget) }}</div>
      </div>
      <div class="stat-card warning">
        <div class="stat-label">{{ t('restocking.plannedSpend') }}</div>
        <div class="stat-value">{{ formatMoney(plannedSpend) }}</div>
      </div>
      <div class="stat-card success">
        <div class="stat-label">{{ t('restocking.remainingBudget') }}</div>
        <div class="stat-value">{{ formatMoney(remainingBudget) }}</div>
      </div>
      <div class="stat-card">
        <div class="stat-label">{{ t('restocking.longestLeadTime') }}</div>
        <div class="stat-value">{{ t('restocking.days', { count: longestLeadTime }) }}</div>
      </div>
    </div>

    <div v-if="orderSuccess" class="success-message" role="status">
      <span>{{ t('restocking.orderPlaced', { id: orderSuccess.id, date: formatDate(orderSuccess.expected_delivery) }) }}</span>
      <router-link to="/orders" class="success-link">{{ t('restocking.viewOrders') }}</router-link>
    </div>
    <div v-if="orderError" class="error" role="alert">{{ orderError }}</div>

    <div class="card">
      <div class="card-header">
        <h3 class="card-title">
          {{ t('restocking.recommendations') }}
          <span v-if="items.length" class="selected-count">
            ({{ t('restocking.selected', { selected: selectedItems.length, total: items.length }) }})
          </span>
        </h3>
        <button
          type="button"
          class="place-order-btn"
          :disabled="selectedItems.length === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? t('restocking.placing') : t('restocking.placeOrder') }}
        </button>
      </div>

      <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else-if="items.length === 0" class="empty-state">{{ t('restocking.empty') }}</div>
      <div v-else class="table-container">
        <table>
          <thead>
            <tr>
              <th class="col-check">
                <input
                  type="checkbox"
                  :checked="allSelected"
                  :aria-label="t('restocking.selectAll')"
                  @change="toggleAll($event.target.checked)"
                />
              </th>
              <th>{{ t('restocking.table.sku') }}</th>
              <th>{{ t('restocking.table.name') }}</th>
              <th>{{ t('restocking.table.category') }}</th>
              <th>{{ t('restocking.table.trend') }}</th>
              <th class="num">{{ t('restocking.table.onHand') }}</th>
              <th class="num">{{ t('restocking.table.forecastedDemand') }}</th>
              <th class="num">{{ t('restocking.table.recommendedQty') }}</th>
              <th class="num">{{ t('restocking.table.unitCost') }}</th>
              <th class="num">{{ t('restocking.table.lineCost') }}</th>
              <th class="num">{{ t('restocking.table.leadTime') }}</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in items" :key="item.sku">
              <td class="col-check">
                <input
                  type="checkbox"
                  :checked="selectedSkus.includes(item.sku)"
                  :aria-label="t('restocking.selectItem', { sku: item.sku })"
                  @change="toggleItem(item.sku, $event.target.checked)"
                />
              </td>
              <td><strong>{{ item.sku }}</strong></td>
              <td>{{ translateProductName(item.name) }}</td>
              <td>{{ item.category }}</td>
              <td>
                <span :class="['badge', item.trend]">{{ t(`trends.${item.trend}`) }}</span>
              </td>
              <td class="num">{{ item.quantity_on_hand.toLocaleString() }}</td>
              <td class="num">{{ item.forecasted_demand.toLocaleString() }}</td>
              <td class="num"><strong>{{ item.recommended_quantity.toLocaleString() }}</strong></td>
              <td class="num">{{ formatMoney(item.unit_cost) }}</td>
              <td class="num"><strong>{{ formatMoney(item.line_cost) }}</strong></td>
              <td class="num">{{ t('restocking.days', { count: item.lead_time_days }) }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

const DEBOUNCE_MS = 250

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, currentLocale, translateProductName } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const budget = ref(25000)
    const items = ref([])
    const selectedSkus = ref([])
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const orderError = ref(null)
    const orderSuccess = ref(null)

    let debounceTimer = null
    let requestSeq = 0

    const formatMoney = (value) => {
      const num = Number(value)
      return `${currencySymbol.value}${(Number.isFinite(num) ? num : 0).toLocaleString()}`
    }

    const formatDate = (dateString) => {
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return dateString
      const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'
      return date.toLocaleDateString(locale, { year: 'numeric', month: 'short', day: 'numeric' })
    }

    const selectedItems = computed(() => {
      return items.value.filter(item => selectedSkus.value.includes(item.sku))
    })

    const allSelected = computed(() => {
      return items.value.length > 0 && selectedItems.value.length === items.value.length
    })

    const plannedSpend = computed(() => {
      return selectedItems.value.reduce((sum, item) => sum + item.line_cost, 0)
    })

    const remainingBudget = computed(() => budget.value - plannedSpend.value)

    const longestLeadTime = computed(() => {
      return selectedItems.value.reduce((max, item) => Math.max(max, item.lead_time_days), 0)
    })

    const loadRecommendations = async () => {
      const seq = ++requestSeq
      try {
        loading.value = true
        error.value = null
        const data = await api.getRestockRecommendations(budget.value)
        if (seq !== requestSeq) return // stale response
        items.value = data.items || []
        selectedSkus.value = items.value.map(item => item.sku)
      } catch (err) {
        if (seq !== requestSeq) return
        items.value = []
        selectedSkus.value = []
        error.value = t('restocking.loadError') + err.message
      } finally {
        if (seq === requestSeq) loading.value = false
      }
    }

    watch(budget, () => {
      clearTimeout(debounceTimer)
      debounceTimer = setTimeout(loadRecommendations, DEBOUNCE_MS)
    })

    const toggleItem = (sku, checked) => {
      if (checked) {
        if (!selectedSkus.value.includes(sku)) selectedSkus.value = [...selectedSkus.value, sku]
      } else {
        selectedSkus.value = selectedSkus.value.filter(s => s !== sku)
      }
    }

    const toggleAll = (checked) => {
      selectedSkus.value = checked ? items.value.map(item => item.sku) : []
    }

    const placeOrder = async () => {
      if (selectedItems.value.length === 0 || submitting.value) return
      submitting.value = true
      orderError.value = null
      orderSuccess.value = null
      try {
        const order = await api.createRestockOrder(
          selectedItems.value.map(item => ({ sku: item.sku, quantity: item.recommended_quantity }))
        )
        orderSuccess.value = order
        // Stock levels changed, so refresh the plan
        clearTimeout(debounceTimer)
        await loadRecommendations()
      } catch (err) {
        const detail = err.response && err.response.data && err.response.data.detail
        const message = typeof detail === 'string' ? detail : (detail ? JSON.stringify(detail) : err.message)
        orderError.value = `${t('restocking.orderError')}: ${message}`
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadRecommendations)
    onBeforeUnmount(() => clearTimeout(debounceTimer))

    return {
      t,
      budget,
      items,
      selectedSkus,
      selectedItems,
      allSelected,
      loading,
      error,
      submitting,
      orderError,
      orderSuccess,
      plannedSpend,
      remainingBudget,
      longestLeadTime,
      formatMoney,
      formatDate,
      toggleItem,
      toggleAll,
      placeOrder,
      translateProductName
    }
  }
}
</script>

<style scoped>
.budget-card {
  padding: 1.5rem;
}

.budget-header {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  margin-bottom: 0.75rem;
}

.budget-header .stat-label {
  margin-bottom: 0;
}

.budget-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-scale {
  display: flex;
  justify-content: space-between;
  margin-top: 0.25rem;
  font-size: 0.813rem;
  color: #64748b;
}

.selected-count {
  font-size: 0.875rem;
  font-weight: 500;
  color: #64748b;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 0.5rem 1.25rem;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  color: #64748b;
  cursor: not-allowed;
}

.success-message {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1rem;
  background: #ecfdf5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 1rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
}

.success-link {
  color: #047857;
  font-weight: 600;
  text-decoration: underline;
  white-space: nowrap;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}

.col-check {
  width: 40px;
}

.num {
  text-align: right;
}
</style>
