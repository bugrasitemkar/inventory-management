<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Success banner -->
      <div v-if="orderSuccess" class="success-banner">
        {{ t('restocking.orderPlaced') }}
      </div>

      <!-- Budget slider card -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetLabel') }}</h3>
        </div>
        <div class="budget-body">
          <div class="budget-amount">{{ currencySymbol }}{{ budget.toLocaleString() }}</div>
          <input
            type="range"
            class="budget-slider"
            min="0"
            max="200000"
            step="1000"
            v-model.number="budget"
          />
          <div class="budget-stats">
            <div class="budget-stat">
              <span class="budget-stat-label">{{ t('restocking.totalCost') }}</span>
              <span class="budget-stat-value">{{ currencySymbol }}{{ totalCost.toLocaleString() }}</span>
            </div>
            <div class="budget-stat">
              <span class="budget-stat-label">{{ t('restocking.remainingBudget') }}</span>
              <span class="budget-stat-value" :class="remainingBudget < 0 ? 'negative' : ''">
                {{ currencySymbol }}{{ remainingBudget.toLocaleString() }}
              </span>
            </div>
          </div>
        </div>
      </div>

      <!-- Stats row -->
      <div class="stats-grid restocking-stats">
        <div class="stat-card">
          <div class="stat-label">Items Selected</div>
          <div class="stat-value">{{ selectedRecommendations.length }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">{{ t('restocking.totalCost') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ totalCost.toLocaleString() }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">{{ t('restocking.remainingBudget') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ remainingBudget.toLocaleString() }}</div>
        </div>
      </div>

      <!-- Recommendations table card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            {{ t('restocking.recommendations') }} ({{ selectedRecommendations.length }})
          </h3>
          <button
            class="place-order-btn"
            :disabled="selectedRecommendations.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? '...' : t('restocking.placeOrder') }}
          </button>
        </div>

        <div v-if="selectedRecommendations.length === 0" class="empty-state">
          {{ t('restocking.recommendationsEmpty') }}
        </div>
        <div v-else class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th class="col-num">{{ t('restocking.table.forecastedDemand') }}</th>
                <th class="col-num">{{ t('restocking.table.quantityToOrder') }}</th>
                <th class="col-num">{{ t('restocking.table.unitCost') }}</th>
                <th class="col-num">{{ t('restocking.table.lineCost') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in selectedRecommendations" :key="item.item_sku">
                <td><code class="sku-code">{{ item.item_sku }}</code></td>
                <td>{{ item.item_name }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-num">{{ item.forecasted_demand.toLocaleString() }}</td>
                <td class="col-num">{{ item.quantity.toLocaleString() }}</td>
                <td class="col-num">
                  {{ currencySymbol }}{{ item.unit_cost.toFixed(2) }}
                  <small v-if="!item.has_inventory_match" class="estimated-note">
                    {{ t('restocking.fallbackCostNote') }}
                  </small>
                </td>
                <td class="col-num"><strong>{{ currencySymbol }}{{ item.line_cost.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const orderSuccess = ref(false)
    const forecasts = ref([])
    const inventoryMap = ref({})  // keyed by sku for O(1) cross-ref lookup
    const budget = ref(50000)

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    // Derive restocking recommendations from demand forecasts cross-referenced with inventory
    const recommendations = computed(() => {
      // Exclude decreasing-trend items — not worth restocking
      const eligible = forecasts.value.filter(f => f.trend !== 'decreasing')

      // Increasing items take priority over stable in greedy budget allocation
      const sorted = [...eligible].sort((a, b) => {
        const priority = { increasing: 0, stable: 1 }
        return priority[a.trend] - priority[b.trend]
      })

      return sorted.map(f => {
        const inv = inventoryMap.value[f.item_sku]
        // Fall back to $50/unit when demand forecast SKU has no matching inventory record
        const unit_cost = inv ? inv.unit_cost : 50.0
        const has_inventory_match = !!inv
        const qty_on_hand = inv ? inv.quantity_on_hand : 0
        // Order only the gap: how many units are needed beyond current stock
        const quantity = inv
          ? Math.max(0, f.forecasted_demand - qty_on_hand)
          : f.forecasted_demand
        const line_cost = quantity * unit_cost
        return { ...f, unit_cost, has_inventory_match, quantity, line_cost }
      }).filter(item => item.quantity > 0)  // skip items already fully stocked
    })

    // Greedy selection: pick items in priority order until budget is exhausted.
    // Items that don't individually fit are skipped (no backtracking).
    const selectedRecommendations = computed(() => {
      let remaining = budget.value
      const selected = []
      for (const item of recommendations.value) {
        if (item.line_cost <= remaining) {
          selected.push(item)
          remaining -= item.line_cost
        }
      }
      return selected
    })

    const totalCost = computed(() =>
      selectedRecommendations.value.reduce((sum, i) => sum + i.line_cost, 0)
    )

    const remainingBudget = computed(() => budget.value - totalCost.value)

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        forecasts.value = forecastsData
        // Build SKU-keyed map for O(1) cross-reference with demand forecasts
        inventoryMap.value = Object.fromEntries(
          inventoryData.map(item => [item.sku, item])
        )
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (selectedRecommendations.value.length === 0) return
      try {
        submitting.value = true
        const orderData = {
          items: selectedRecommendations.value.map(item => ({
            sku: item.item_sku,
            name: item.item_name,
            quantity: item.quantity,
            unit_cost: item.unit_cost
          })),
          total_value: Math.round(totalCost.value * 100) / 100
        }
        await api.createRestockingOrder(orderData)
        orderSuccess.value = true
        // Auto-hide success banner after 5 seconds
        setTimeout(() => { orderSuccess.value = false }, 5000)
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      t,
      loading,
      error,
      submitting,
      orderSuccess,
      budget,
      currencySymbol,
      recommendations,
      selectedRecommendations,
      totalCost,
      remainingBudget,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  /* Uses global .loading, .error, .card, .card-header, .card-title, .stats-grid, .stat-card, .badge from App.vue */
}

/* Budget card */
.budget-body {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.budget-amount {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  cursor: pointer;
  height: 6px;
}

.budget-stats {
  display: flex;
  gap: 2rem;
}

.budget-stat {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.budget-stat-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: #64748b;
}

.budget-stat-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-stat-value.negative {
  color: #dc2626;
}

/* Stats row — 3 columns */
.restocking-stats {
  grid-template-columns: repeat(3, 1fr);
}

/* Recommendations table */
.restocking-table {
  width: 100%;
  border-collapse: collapse;
}

.col-num {
  text-align: right;
}

.sku-code {
  font-family: monospace;
  font-size: 0.813rem;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
  color: #475569;
}

.estimated-note {
  font-size: 0.72rem;
  color: #94a3b8;
  font-style: italic;
  display: block;
}

/* Empty state */
.empty-state {
  padding: 2.5rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

/* Place Order button */
.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 0.5rem 1.25rem;
  font-weight: 600;
  font-size: 0.875rem;
  cursor: pointer;
  transition: background 0.15s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Success banner */
.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 1rem;
  color: #065f46;
  font-weight: 500;
  margin-bottom: 1rem;
}
</style>
