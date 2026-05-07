<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Recommend items to restock based on demand forecast and available budget</p>
    </div>

    <div v-if="loading" class="loading">Loading recommendations...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Card -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Budget Configuration</h3>
        </div>
        <div class="budget-controls">
          <div class="budget-label-row">
            <span class="budget-label">Available Budget</span>
            <span class="budget-value">{{ formatCurrency(budget) }}</span>
          </div>
          <div class="slider-wrapper">
            <input
              type="range"
              class="budget-slider"
              min="0"
              max="500000"
              step="5000"
              v-model.number="budget"
            />
            <div class="slider-endpoints">
              <span>$0</span>
              <span>$500K</span>
            </div>
          </div>
        </div>
      </div>

      <!-- Recommendations Card -->
      <div class="card recommendations-card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ recommendedItems.length }})</h3>
        </div>

        <!-- Budget Usage Bar -->
        <div class="budget-usage">
          <div class="usage-bar-track">
            <div
              class="usage-bar-fill"
              :style="{ width: budgetUsagePercent + '%' }"
              :class="{ 'over-limit': budgetUsagePercent >= 100 }"
            ></div>
          </div>
          <div class="usage-label">
            {{ formatCurrency(spentBudget) }} of {{ formatCurrency(budget) }}
            <span class="usage-percent">({{ budgetUsagePercent }}%)</span>
          </div>
        </div>

        <!-- Confirmation State (post-order) -->
        <div v-if="orderConfirmation" class="order-confirmation">
          <div class="confirmation-icon">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <circle cx="12" cy="12" r="10"/>
              <polyline points="9,12 11,14 15,10"/>
            </svg>
          </div>
          <h4 class="confirmation-title">Order Submitted Successfully</h4>
          <div class="confirmation-details">
            <div class="confirmation-row">
              <span class="confirmation-label">Order Number</span>
              <span class="confirmation-order-number">{{ orderConfirmation.order_number }}</span>
            </div>
            <div class="confirmation-row">
              <span class="confirmation-label">Items</span>
              <span>{{ orderConfirmation.items.length }} items</span>
            </div>
            <div class="confirmation-row">
              <span class="confirmation-label">Total Cost</span>
              <span class="confirmation-cost">{{ formatCurrency(orderConfirmation.total_cost) }}</span>
            </div>
            <div class="confirmation-row">
              <span class="confirmation-label">Estimated Delivery</span>
              <span>{{ formatDate(orderConfirmation.estimated_delivery) }}</span>
            </div>
            <div class="confirmation-row">
              <span class="confirmation-label">Lead Time</span>
              <span>{{ orderConfirmation.lead_time_days }}-day lead time</span>
            </div>
          </div>
          <button class="btn-secondary" @click="resetOrder">Place Another Order</button>
        </div>

        <!-- Item List State -->
        <div v-else>
          <div v-if="recommendedItems.length === 0 && budget === 0" class="empty-state">
            Adjust the budget slider above to see recommended items.
          </div>
          <div v-else-if="recommendedItems.length === 0" class="empty-state">
            No items fit within the current budget.
          </div>

          <!-- Items within budget -->
          <div v-if="recommendedItems.length > 0" class="item-list">
            <div
              v-for="item in recommendedItems"
              :key="item.id"
              class="item-row"
            >
              <span :class="['badge', item.trend]">{{ item.trend.toUpperCase() }}</span>
              <span class="item-sku">{{ item.item_sku }}</span>
              <span class="item-name">{{ item.item_name }}</span>
              <span class="item-cost-detail">
                {{ item.restock_quantity.toLocaleString() }} units
                &times; {{ formatCurrency(item.unit_cost) }}
                = <strong>{{ formatCurrency(item.estimated_cost) }}</strong>
              </span>
            </div>
          </div>

          <!-- Over-budget items -->
          <div v-if="overBudgetItems.length > 0" class="over-budget-section">
            <div class="over-budget-header">Over Budget — Not Included</div>
            <div
              v-for="item in overBudgetItems"
              :key="item.id"
              class="item-row item-row-dimmed"
            >
              <span :class="['badge', item.trend, 'badge-dimmed']">{{ item.trend.toUpperCase() }}</span>
              <span class="item-sku">{{ item.item_sku }}</span>
              <span class="item-name">{{ item.item_name }}</span>
              <span class="item-cost-detail">
                {{ item.restock_quantity.toLocaleString() }} units
                &times; {{ formatCurrency(item.unit_cost) }}
                = <strong>{{ formatCurrency(item.estimated_cost) }}</strong>
              </span>
            </div>
          </div>

          <!-- Inline error -->
          <div v-if="submitError" class="submit-error">{{ submitError }}</div>

          <!-- Action Bar -->
          <div class="action-bar">
            <button
              class="btn-primary"
              :disabled="submitting || recommendedItems.length === 0"
              @click="placeOrder"
            >
              <span v-if="submitting">Submitting...</span>
              <span v-else>Place Order</span>
            </button>
            <span class="action-summary">
              {{ recommendedItems.length }} items &middot; {{ formatCurrency(spentBudget) }} total
            </span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const recommendations = ref([])
    const loading = ref(true)
    const error = ref(null)
    const budget = ref(100000)
    const submitting = ref(false)
    const submitError = ref(null)
    const orderConfirmation = ref(null)

    const recommendedItems = computed(() => {
      let runningTotal = 0
      const included = []
      for (const item of recommendations.value) {
        if (runningTotal + item.estimated_cost <= budget.value) {
          included.push(item)
          runningTotal += item.estimated_cost
        }
      }
      return included
    })

    const overBudgetItems = computed(() => {
      const includedIds = new Set(recommendedItems.value.map(i => i.id))
      return recommendations.value.filter(i => !includedIds.has(i.id))
    })

    const spentBudget = computed(() => {
      return recommendedItems.value.reduce((sum, item) => sum + item.estimated_cost, 0)
    })

    const budgetUsagePercent = computed(() => {
      if (budget.value === 0) return 0
      return Math.min(100, Math.round((spentBudget.value / budget.value) * 100))
    })

    const formatCurrency = (value) => {
      if (value === undefined || value === null) return '$0'
      return '$' + value.toLocaleString('en-US', { maximumFractionDigits: 0 })
    }

    const formatDate = (dateString) => {
      if (!dateString) return '—'
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return '—'
      return date.toLocaleDateString('en-US', {
        year: 'numeric',
        month: 'short',
        day: 'numeric'
      })
    }

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        recommendations.value = await api.getRestockRecommendations()
      } catch (err) {
        error.value = 'Failed to load restock recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitting.value = true
      submitError.value = null
      try {
        const result = await api.createRestockOrder({
          items: recommendedItems.value,
          budget: budget.value
        })
        orderConfirmation.value = result
      } catch (err) {
        submitError.value = 'Failed to submit order. Please try again.'
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    const resetOrder = () => {
      orderConfirmation.value = null
      submitError.value = null
    }

    onMounted(loadRecommendations)

    return {
      recommendations,
      loading,
      error,
      budget,
      submitting,
      submitError,
      orderConfirmation,
      recommendedItems,
      overBudgetItems,
      spentBudget,
      budgetUsagePercent,
      formatCurrency,
      formatDate,
      placeOrder,
      resetOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

/* Budget Card */
.budget-card {
  margin-bottom: 1.25rem;
}

.budget-controls {
  padding: 0.25rem 0;
}

.budget-label-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.budget-label {
  font-size: 0.938rem;
  font-weight: 600;
  color: #475569;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  font-size: 0.75rem;
}

.budget-value {
  font-size: 1.75rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.slider-wrapper {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  appearance: none;
  -webkit-appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  appearance: none;
  -webkit-appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  border: 2px solid #fff;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  border: 2px solid #fff;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
}

.slider-endpoints {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #94a3b8;
  font-weight: 500;
}

/* Budget Usage */
.budget-usage {
  margin-bottom: 1.25rem;
}

.usage-bar-track {
  height: 8px;
  background: #f1f5f9;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 0.5rem;
}

.usage-bar-fill {
  height: 100%;
  background: #3b82f6;
  border-radius: 4px;
  transition: width 0.3s ease;
}

.usage-bar-fill.over-limit {
  background: #ef4444;
}

.usage-label {
  font-size: 0.875rem;
  color: #64748b;
}

.usage-percent {
  color: #94a3b8;
  margin-left: 0.25rem;
}

/* Item List */
.item-list {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.item-row {
  display: flex;
  align-items: center;
  gap: 0.875rem;
  padding: 0.75rem 1rem;
  background: #f8fafc;
  border-radius: 8px;
  border: 1px solid #e2e8f0;
}

.item-row-dimmed {
  opacity: 0.45;
  background: #f1f5f9;
}

.item-sku {
  font-family: 'Courier New', Courier, monospace;
  font-size: 0.813rem;
  font-weight: 600;
  color: #475569;
  min-width: 80px;
}

.item-name {
  flex: 1;
  font-size: 0.875rem;
  color: #0f172a;
  font-weight: 500;
}

.item-cost-detail {
  font-size: 0.813rem;
  color: #64748b;
  white-space: nowrap;
}

/* Over-budget section */
.over-budget-section {
  margin-top: 1.25rem;
  margin-bottom: 1rem;
}

.over-budget-header {
  font-size: 0.75rem;
  font-weight: 600;
  color: #94a3b8;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 0.5rem;
  padding-left: 0.25rem;
}

.badge-dimmed {
  opacity: 0.6;
}

/* Empty state */
.empty-state {
  text-align: center;
  padding: 2.5rem;
  color: #94a3b8;
  font-size: 0.938rem;
}

/* Action Bar */
.action-bar {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding-top: 1.25rem;
  border-top: 1px solid #e2e8f0;
  margin-top: 0.5rem;
}

.btn-primary {
  background: #3b82f6;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #2563eb;
}

.btn-primary:disabled {
  background: #93c5fd;
  cursor: not-allowed;
}

.btn-secondary {
  background: white;
  color: #3b82f6;
  border: 1px solid #bfdbfe;
  padding: 0.625rem 1.5rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
  margin-top: 1.25rem;
}

.btn-secondary:hover {
  background: #eff6ff;
  border-color: #93c5fd;
}

.action-summary {
  font-size: 0.875rem;
  color: #64748b;
}

/* Inline error */
.submit-error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: 0.75rem 1rem;
  border-radius: 8px;
  font-size: 0.875rem;
  margin-bottom: 1rem;
}

/* Order Confirmation */
.order-confirmation {
  padding: 1.5rem;
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  border-radius: 10px;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
}

.confirmation-icon {
  width: 40px;
  height: 40px;
  color: #059669;
  margin-bottom: 0.75rem;
}

.confirmation-title {
  font-size: 1.125rem;
  font-weight: 700;
  color: #064e3b;
  margin-bottom: 1rem;
}

.confirmation-details {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  width: 100%;
  margin-bottom: 0.25rem;
}

.confirmation-row {
  display: flex;
  gap: 1rem;
  font-size: 0.875rem;
}

.confirmation-label {
  font-weight: 600;
  color: #065f46;
  min-width: 150px;
}

.confirmation-order-number {
  font-family: 'Courier New', Courier, monospace;
  font-weight: 700;
  color: #0f172a;
}

.confirmation-cost {
  font-weight: 700;
  color: #0f172a;
}
</style>
