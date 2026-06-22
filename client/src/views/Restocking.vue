<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Recommended items based on stock levels and demand trends</p>
    </div>

    <!-- Budget control card -->
    <div class="card budget-card">
      <div class="card-header">
        <h3 class="card-title">Available Budget</h3>
        <span class="budget-display">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
      </div>
      <div class="budget-slider-wrapper">
        <input type="range" min="0" max="50000" step="500" v-model.number="budget" class="budget-slider" />
        <div class="slider-labels">
          <span>{{ currencySymbol }}0</span>
          <span>{{ currencySymbol }}50,000</span>
        </div>
      </div>
      <div class="budget-summary">
        <span>Estimated Total: <strong>{{ currencySymbol }}{{ totalEstimatedCost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong></span>
        <span class="sep">·</span>
        <span>Budget Used: <strong>{{ budgetUtilization }}%</strong></span>
        <span class="sep">·</span>
        <span>Items Selected: <strong>{{ recommendedItems.length }}</strong></span>
      </div>
    </div>

    <!-- Success banner -->
    <div v-if="submittedOrder" class="success-banner">
      <span>Order <strong>{{ submittedOrder.order_number }}</strong> placed. Expected delivery: <strong>{{ formatDate(submittedOrder.expected_delivery) }}</strong></span>
      <button class="dismiss-btn" @click="submittedOrder = null">Dismiss</button>
    </div>

    <!-- Recommended items card -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Recommended Items ({{ recommendedItems.length }})</h3>
        <button
          class="place-order-btn"
          :disabled="recommendedItems.length === 0 || isSubmitting"
          @click="placeOrder"
        >
          {{ isSubmitting ? 'Placing Order...' : 'Place Order' }}
        </button>
      </div>

      <div v-if="loading" class="loading">Loading restocking data...</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else-if="recommendedItems.length === 0" class="empty-state">
        No items recommended within the current budget. Try increasing the budget or adjusting filters.
      </div>
      <div v-else class="table-container">
        <table>
          <thead>
            <tr>
              <th>Priority</th>
              <th>SKU</th>
              <th>Item Name</th>
              <th>Category</th>
              <th>Warehouse</th>
              <th>On Hand</th>
              <th>Reorder Point</th>
              <th>Trend</th>
              <th>Suggested Qty</th>
              <th>Unit Cost</th>
              <th>Estimated Cost</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in recommendedItems" :key="item.id">
              <td>
                <span :class="['badge', item.below_reorder_point ? 'danger' : 'warning']">
                  {{ item.below_reorder_point ? 'Critical' : 'Forecast' }}
                </span>
              </td>
              <td><code>{{ item.sku }}</code></td>
              <td>{{ item.name }}</td>
              <td>{{ item.category }}</td>
              <td>{{ item.warehouse }}</td>
              <td>{{ item.quantity_on_hand }}</td>
              <td>{{ item.reorder_point }}</td>
              <td><span :class="['badge', item.trend]">{{ item.trend }}</span></td>
              <td><strong>{{ item.suggested_quantity }}</strong></td>
              <td>{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
              <td><strong>{{ currencySymbol }}{{ item.estimated_cost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong></td>
            </tr>
          </tbody>
        </table>
        <p class="table-note">Items sorted by urgency. Below-reorder-point items appear first.</p>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()
    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    const loading = ref(true)
    const error = ref(null)
    const allCandidates = ref([])
    const budget = ref(10000)
    const isSubmitting = ref(false)
    const submittedOrder = ref(null)

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    // Two-group sort: below-reorder-point items first (by urgency), then increasing-trend items
    const sortedCandidates = computed(() => {
      const belowReorder = allCandidates.value
        .filter(i => i.below_reorder_point)
        .sort((a, b) => a.urgency_ratio - b.urgency_ratio)

      const increasing = allCandidates.value
        .filter(i => !i.below_reorder_point && i.trend === 'increasing')
        .sort((a, b) => (b.forecasted_demand - b.current_demand) - (a.forecasted_demand - a.current_demand))

      return [...belowReorder, ...increasing]
    })

    // Greedy best-effort selection: continue past items that don't fit
    const recommendedItems = computed(() => {
      let remaining = budget.value
      return sortedCandidates.value.filter(item => {
        if (item.estimated_cost <= remaining) {
          remaining -= item.estimated_cost
          return true
        }
        return false
      })
    })

    const totalEstimatedCost = computed(() =>
      recommendedItems.value.reduce((sum, i) => sum + i.estimated_cost, 0)
    )

    const budgetUtilization = computed(() =>
      budget.value > 0 ? Math.round((totalEstimatedCost.value / budget.value) * 100) : 0
    )

    const loadCandidates = async () => {
      try {
        loading.value = true
        error.value = null
        const filters = getCurrentFilters()
        allCandidates.value = await api.getRestockingCandidates(filters)
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    // Only watch location + category — endpoint does not support period/status filters
    watch([selectedLocation, selectedCategory], loadCandidates)
    onMounted(loadCandidates)

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0 || isSubmitting.value) return
      isSubmitting.value = true
      try {
        const filters = getCurrentFilters()
        const orderData = {
          items: recommendedItems.value.map(i => ({
            sku: i.sku,
            name: i.name,
            quantity: i.suggested_quantity,
            unit_price: i.unit_cost
          })),
          total_value: Math.round(totalEstimatedCost.value * 100) / 100,
          warehouse: filters.warehouse !== 'all' ? filters.warehouse : null,
          category: filters.category !== 'all' ? filters.category : null
        }
        submittedOrder.value = await api.createRestockingOrder(orderData)
      } catch (err) {
        error.value = 'Failed to place restocking order: ' + err.message
      } finally {
        isSubmitting.value = false
      }
    }

    const formatDate = (dateString) => {
      return new Date(dateString).toLocaleDateString('en-US', {
        year: 'numeric', month: 'short', day: 'numeric'
      })
    }

    return {
      loading,
      error,
      budget,
      isSubmitting,
      submittedOrder,
      recommendedItems,
      totalEstimatedCost,
      budgetUtilization,
      currencySymbol,
      placeOrder,
      formatDate
    }
  }
}
</script>

<style scoped>
.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2563eb;
}

.budget-slider-wrapper {
  padding: 0.5rem 0;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  cursor: pointer;
  height: 6px;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.8rem;
  color: #64748b;
  margin-top: 0.25rem;
}

.budget-summary {
  display: flex;
  gap: 1rem;
  flex-wrap: wrap;
  font-size: 0.875rem;
  color: #64748b;
  margin-top: 0.75rem;
  padding-top: 0.75rem;
  border-top: 1px solid #e2e8f0;
}

.sep {
  color: #cbd5e1;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  border-radius: 8px;
  padding: 0.875rem 1.25rem;
  margin-bottom: 1.25rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.9rem;
}

.dismiss-btn {
  background: transparent;
  border: 1px solid #6ee7b7;
  color: #065f46;
  border-radius: 5px;
  padding: 0.25rem 0.75rem;
  cursor: pointer;
  font-size: 0.8rem;
}

.dismiss-btn:hover {
  background: #a7f3d0;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 0.5rem 1.25rem;
  font-weight: 600;
  font-size: 0.875rem;
  cursor: pointer;
  transition: background 0.2s;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}

.table-note {
  font-size: 0.78rem;
  color: #94a3b8;
  margin-top: 0.75rem;
  text-align: right;
}

code {
  font-family: monospace;
  font-size: 0.82rem;
  background: #f1f5f9;
  padding: 0.1rem 0.35rem;
  border-radius: 3px;
  color: #334155;
}
</style>
