<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set a budget and get recommended restock quantities based on current stock levels and forecasted demand.</p>
    </div>

    <div class="card budget-card">
      <div class="card-header">
        <h3 class="card-title">Available Budget</h3>
      </div>
      <div class="budget-body">
        <div class="budget-display">{{ currencySymbol }}{{ budget.toLocaleString() }}</div>
        <input
          type="range"
          class="budget-slider"
          min="0"
          max="1000000"
          step="25000"
          v-model.number="budget"
        />
        <div class="budget-endpoints">
          <span>{{ currencySymbol }}0</span>
          <span>{{ currencySymbol }}1,000,000</span>
        </div>
      </div>
    </div>

    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Recommended Restocks ({{ recommendations.length }})</h3>
      </div>

      <div class="restock-summary" v-if="!loading && recommendations.length > 0">
        <div class="summary-pill">
          <div class="summary-pill-label">Total Cost</div>
          <div class="summary-pill-value">{{ currencySymbol }}{{ totalCost.toLocaleString() }}</div>
        </div>
        <div class="summary-pill">
          <div class="summary-pill-label">Remaining Budget</div>
          <div class="summary-pill-value">{{ currencySymbol }}{{ remaining.toLocaleString() }}</div>
        </div>
        <div class="summary-pill">
          <div class="summary-pill-label">Items</div>
          <div class="summary-pill-value">{{ recommendations.length }}</div>
        </div>
      </div>

      <div v-if="loading" class="loading">Calculating recommendations...</div>
      <div v-else-if="recommendations.length === 0" class="empty-state">
        No items need restocking within this budget. Try increasing your budget.
      </div>
      <div v-else class="table-container">
        <table class="restock-table">
          <thead>
            <tr>
              <th>SKU</th>
              <th>Item</th>
              <th>Category</th>
              <th>Warehouse</th>
              <th>Current Stock</th>
              <th>Forecasted Demand</th>
              <th>Gap</th>
              <th>Qty to Order</th>
              <th>Unit Cost</th>
              <th>Line Cost</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="r in recommendations" :key="r.sku">
              <td><strong>{{ r.sku }}</strong></td>
              <td>{{ r.name }}</td>
              <td>{{ r.category }}</td>
              <td>{{ r.warehouse }}</td>
              <td>{{ r.current_stock }}</td>
              <td>{{ r.forecasted_demand }}</td>
              <td>
                <span class="gap-value">{{ r.gap }}</span>
              </td>
              <td><strong>{{ r.recommended_quantity }}</strong></td>
              <td>{{ currencySymbol }}{{ r.unit_cost.toLocaleString() }}</td>
              <td><strong>{{ currencySymbol }}{{ r.line_cost.toLocaleString() }}</strong></td>
            </tr>
          </tbody>
        </table>
      </div>

      <div class="order-action-row">
        <div v-if="successMessage" class="success-banner">
          {{ successMessage }}
        </div>
        <div v-if="submitError" class="error">{{ submitError }}</div>
        <button
          class="btn-primary"
          :disabled="recommendations.length === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? 'Placing Order...' : 'Place Order' }}
        </button>
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
    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '\xa5' : '$')

    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    const budget = ref(250000)
    const recommendations = ref([])
    const loading = ref(false)
    const totalCost = ref(0)
    const remaining = ref(0)

    const submitting = ref(false)
    const successMessage = ref('')
    const submitError = ref('')

    let successTimer = null

    const loadRecommendations = async () => {
      loading.value = true
      submitError.value = ''
      try {
        const filters = getCurrentFilters()
        const data = await api.getRestockRecommendations(budget.value, filters)
        recommendations.value = data.recommendations
        totalCost.value = data.total_cost
        remaining.value = data.remaining_budget
      } catch (err) {
        console.error('Failed to load restock recommendations:', err)
        recommendations.value = []
        totalCost.value = 0
        remaining.value = budget.value
      } finally {
        loading.value = false
      }
    }

    watch(budget, loadRecommendations)
    watch([selectedLocation, selectedCategory], loadRecommendations)

    const placeOrder = async () => {
      if (recommendations.value.length === 0 || submitting.value) return
      submitting.value = true
      submitError.value = ''
      successMessage.value = ''
      try {
        const result = await api.submitRestockOrder({
          budget: budget.value,
          items: recommendations.value.map(r => ({
            sku: r.sku,
            name: r.name,
            category: r.category,
            warehouse: r.warehouse,
            quantity: r.recommended_quantity,
            unit_cost: r.unit_cost
          }))
        })
        const deliveryDate = new Date(result.expected_delivery)
        const formattedDelivery = isNaN(deliveryDate.getTime())
          ? result.expected_delivery
          : deliveryDate.toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' })
        successMessage.value = `Order ${result.order_number} placed. Expected delivery: ${formattedDelivery} (${result.lead_time_days} day lead time).`
        if (successTimer) clearTimeout(successTimer)
        successTimer = setTimeout(() => {
          successMessage.value = ''
        }, 5000)
        await loadRecommendations()
      } catch (err) {
        console.error('Failed to submit restock order:', err)
        submitError.value = 'Failed to place order. Please try again.'
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadRecommendations)

    return {
      currencySymbol,
      budget,
      recommendations,
      loading,
      totalCost,
      remaining,
      submitting,
      successMessage,
      submitError,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.5rem;
}

.budget-body {
  padding: 1.25rem 1.5rem 1.5rem;
}

.budget-display {
  font-size: 2.25rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 1rem;
  letter-spacing: -0.02em;
}

.budget-slider {
  width: 100%;
  height: 6px;
  appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #fff;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #fff;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
}

.budget-slider::-webkit-slider-runnable-track {
  height: 6px;
  background: #e2e8f0;
  border-radius: 3px;
}

.budget-endpoints {
  display: flex;
  justify-content: space-between;
  margin-top: 0.375rem;
  font-size: 0.8125rem;
  color: #64748b;
}

.restock-summary {
  display: flex;
  gap: 1rem;
  padding: 1rem 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.summary-pill {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 0.625rem 1rem;
  min-width: 140px;
}

.summary-pill-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 0.25rem;
}

.summary-pill-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.empty-state {
  padding: 2.5rem 1.5rem;
  text-align: center;
  color: #64748b;
  font-size: 0.9375rem;
}

.restock-table {
  table-layout: auto;
  width: 100%;
}

.gap-value {
  color: #ef4444;
  font-weight: 600;
}

.order-action-row {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 0.75rem;
  padding: 1.25rem 1.5rem;
  border-top: 1px solid #e2e8f0;
}

.success-banner {
  width: 100%;
  background: #d1fae5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 1rem;
  border-radius: 8px;
  font-size: 0.9375rem;
  font-weight: 500;
}

.btn-primary {
  background: #2563eb;
  color: #fff;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-size: 0.9375rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
