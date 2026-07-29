<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Budget Slider -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetSlider') }}</h3>
        </div>
        <div class="budget-controls">
          <input
            type="range"
            class="budget-slider"
            v-model.number="budget"
            :min="1000"
            :max="50000"
            :step="1000"
          />
          <div class="budget-labels">
            <span class="budget-label-item">
              <span class="budget-label-name">{{ t('restocking.availableBudget') }}:</span>
              <span class="budget-label-value">{{ formatCurrency(budget) }}</span>
            </span>
            <span class="budget-label-item">
              <span class="budget-label-name">{{ t('restocking.estimatedOrderCost') }}:</span>
              <span class="budget-label-value cost">{{ formatCurrency(totalCost) }}</span>
            </span>
            <span class="budget-label-item">
              <span class="budget-label-name">{{ t('restocking.remaining') }}:</span>
              <span :class="['budget-label-value', remaining >= 0 ? 'remaining-positive' : 'remaining-negative']">
                {{ formatCurrency(remaining) }}
              </span>
            </span>
          </div>
        </div>
      </div>

      <!-- Stats Grid -->
      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.selectedItems') }}</div>
          <div class="stat-value">{{ selectedItems.length }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">{{ t('restocking.exceedsBudget') }}</div>
          <div class="stat-value">{{ recommendations.length - selectedItems.length }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('restocking.estimatedOrderCost') }}</div>
          <div class="stat-value total-cost-value">{{ formatCurrency(totalCost) }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('restocking.remaining') }}</div>
          <div :class="['stat-value', remaining >= 0 ? 'remaining-positive' : 'remaining-negative']">
            {{ formatCurrency(remaining) }}
          </div>
        </div>
      </div>

      <!-- Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            {{ t('restocking.recommendedItems') }}
            <span class="item-count">({{ recommendations.length }})</span>
          </h3>
          <div class="card-actions">
            <span v-if="successMessage" class="badge info success-badge">
              {{ successMessage }}
            </span>
            <button
              class="place-order-btn"
              :disabled="selectedItems.length === 0 || orderLoading"
              @click="placeOrder"
            >
              {{ orderLoading ? t('common.loading') : t('restocking.placeOrder') }}
            </button>
          </div>
        </div>

        <div v-if="recommendations.length === 0" class="empty-state">
          {{ t('restocking.noItems') }}
        </div>
        <div v-else class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.category') }}</th>
                <th>{{ t('restocking.table.warehouse') }}</th>
                <th class="col-number">{{ t('restocking.table.inStock') }}</th>
                <th class="col-number">{{ t('restocking.table.reorderPoint') }}</th>
                <th class="col-number">{{ t('restocking.table.quantityToOrder') }}</th>
                <th class="col-number">{{ t('restocking.table.unitCost') }}</th>
                <th class="col-number">{{ t('restocking.table.lineTotal') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.sku"
                :class="['restock-row', isItemSelected(item) ? 'row-selected' : 'row-excluded']"
              >
                <td>{{ item.name }}</td>
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.category }}</td>
                <td>{{ item.warehouse }}</td>
                <td class="col-number">{{ item.quantity_on_hand }}</td>
                <td class="col-number">{{ item.reorder_point }}</td>
                <td class="col-number"><strong>{{ item.quantity_to_order }}</strong></td>
                <td class="col-number">{{ formatCurrency(item.unit_cost) }}</td>
                <td class="col-number">
                  <strong>{{ formatCurrency(item.line_cost) }}</strong>
                  <span v-if="!isItemSelected(item)" class="exceeds-label">
                    {{ t('restocking.exceedsBudget') }}
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted, watch } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t } = useI18n()
    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    const loading = ref(true)
    const error = ref(null)
    const recommendations = ref([])
    const budget = ref(10000)
    const successMessage = ref(null)
    const orderLoading = ref(false)

    // Greedy selection: walk recommendations sorted by line_cost descending (backend provides this order).
    // Include each item while the running total stays within budget.
    const selectedItems = computed(() => {
      let running = 0
      const selected = []
      for (const item of recommendations.value) {
        if (running + item.line_cost <= budget.value) {
          running += item.line_cost
          selected.push(item)
        }
      }
      return selected
    })

    const totalCost = computed(() =>
      selectedItems.value.reduce((sum, item) => sum + item.line_cost, 0)
    )

    const remaining = computed(() => budget.value - totalCost.value)

    const isItemSelected = (item) => {
      return selectedItems.value.some(s => s.sku === item.sku)
    }

    const formatCurrency = (value) => {
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD' })
    }

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const filters = getCurrentFilters()
        recommendations.value = await api.getRestockingRecommendations(filters)
      } catch (err) {
        error.value = 'Failed to load restocking recommendations: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (selectedItems.value.length === 0 || orderLoading.value) return

      orderLoading.value = true
      successMessage.value = null
      error.value = null

      // Derive warehouse and category from selection (single value or "Multiple")
      const warehouses = [...new Set(selectedItems.value.map(i => i.warehouse))]
      const categories = [...new Set(selectedItems.value.map(i => i.category))]
      const warehouse = warehouses.length === 1 ? warehouses[0] : 'Multiple'
      const category = categories.length === 1 ? categories[0] : 'Multiple'

      const orderPayload = {
        items: selectedItems.value.map(item => ({
          sku: item.sku,
          name: item.name,
          quantity: item.quantity_to_order,
          unit_price: item.unit_cost
        })),
        warehouse,
        category
      }

      try {
        await api.placeRestockingOrder(orderPayload)
        successMessage.value = t('restocking.orderPlaced')
        // Reload recommendations after a successful order
        await loadRecommendations()
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        orderLoading.value = false
      }
    }

    watch([selectedLocation, selectedCategory], () => {
      loadRecommendations()
    })

    onMounted(loadRecommendations)

    return {
      t,
      loading,
      error,
      recommendations,
      budget,
      successMessage,
      orderLoading,
      selectedItems,
      totalCost,
      remaining,
      isItemSelected,
      formatCurrency,
      placeOrder
    }
  }
}
</script>

<style scoped>
/* Budget card */
.budget-card {
  margin-bottom: 1.25rem;
}

.budget-controls {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  appearance: auto;
  cursor: pointer;
  accent-color: #2563eb;
}

.budget-labels {
  display: flex;
  gap: 2.5rem;
  flex-wrap: wrap;
}

.budget-label-item {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.budget-label-name {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-label-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-label-value.cost {
  color: #2563eb;
}

.budget-label-value.remaining-positive {
  color: #059669;
}

.budget-label-value.remaining-negative {
  color: #dc2626;
}

/* Stat card overrides for currency values */
.total-cost-value {
  font-size: 1.5rem;
  color: #2563eb;
}

.remaining-positive {
  color: #059669;
}

.remaining-negative {
  color: #dc2626;
}

/* Card actions (header right side) */
.card-actions {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.item-count {
  font-size: 0.875rem;
  font-weight: 400;
  color: #64748b;
  margin-left: 0.375rem;
}

.success-badge {
  font-size: 0.813rem;
}

/* Place Order button */
.place-order-btn {
  padding: 0.5rem 1.25rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease, opacity 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

/* Table */
.restocking-table {
  width: 100%;
  border-collapse: collapse;
}

.col-number {
  text-align: right;
}

/* Row state transitions */
.restock-row {
  transition: background-color 0.2s ease, opacity 0.2s ease, color 0.2s ease;
}

.row-selected {
  /* Inherits default tbody row styling from App.vue */
}

.row-excluded {
  opacity: 0.45;
  background: #f8fafc;
  color: #94a3b8;
}

.row-excluded strong {
  color: #94a3b8;
}

.exceeds-label {
  display: block;
  font-size: 0.688rem;
  font-weight: 600;
  color: #dc2626;
  text-transform: uppercase;
  letter-spacing: 0.025em;
  margin-top: 0.125rem;
}

/* Empty state */
.empty-state {
  padding: 3rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}
</style>
