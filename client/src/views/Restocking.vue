<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Budget Slider Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budget') }}</h3>
          <span class="budget-display">{{ currencySymbol }}{{ formatCurrency(budget) }}</span>
        </div>
        <div class="slider-row">
          <span class="slider-bound">{{ currencySymbol }}1,000</span>
          <input
            type="range"
            min="1000"
            max="500000"
            step="1000"
            v-model.number="budget"
            class="budget-slider"
          />
          <span class="slider-bound">{{ currencySymbol }}500,000</span>
        </div>
      </div>

      <!-- Recommendations Table Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }}</h3>
        </div>

        <div v-if="recommendations.length === 0" class="no-items">
          {{ t('restocking.noItems') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th class="col-check"></th>
                <th>SKU</th>
                <th>{{ t('demand.table.itemName') }}</th>
                <th>{{ t('restocking.trend') }}</th>
                <th>{{ t('restocking.quantity') }}</th>
                <th>{{ t('restocking.unitCost') }}</th>
                <th>{{ t('restocking.lineTotal') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.id"
                :class="getRowClass(item)"
              >
                <td class="col-check">
                  <input
                    v-if="item.quantity_to_restock > 0"
                    type="checkbox"
                    :checked="effectiveSelected(item)"
                    @change="toggleItem(item)"
                    class="row-checkbox"
                  />
                  <span v-else class="no-check-placeholder"></span>
                </td>
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ t(`trends.${item.trend}`) }}</span>
                </td>
                <td>
                  <span v-if="item.quantity_to_restock === 0" class="muted-text">No gap</span>
                  <span v-else>{{ item.quantity_to_restock }}</span>
                </td>
                <td>{{ currencySymbol }}{{ formatCurrency(item.unit_cost) }}</td>
                <td>
                  <span class="line-total-cell">
                    {{ currencySymbol }}{{ formatCurrency(item.line_total) }}
                    <span
                      v-if="isOverBudget(item)"
                      class="badge danger over-budget-badge"
                    >{{ t('restocking.overBudget') }}</span>
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Summary Bar -->
        <div class="summary-bar">
          <div class="summary-stats">
            <span class="summary-item">
              {{ t('restocking.selectedSummary', { selected: selectedItems.length, total: recommendations.length }) }}
            </span>
            <span class="summary-separator">·</span>
            <span class="summary-item">
              {{ t('restocking.budgetUsed') }}: <strong>{{ currencySymbol }}{{ formatCurrency(budgetUsed) }}</strong>
            </span>
            <span class="summary-separator">·</span>
            <span class="summary-item" :class="{ 'over-budget-text': budgetRemaining < 0 }">
              {{ t('restocking.remaining') }}: <strong>{{ currencySymbol }}{{ formatCurrency(Math.abs(budgetRemaining)) }}{{ budgetRemaining < 0 ? ' over' : '' }}</strong>
            </span>
          </div>
          <button
            class="place-order-btn"
            :disabled="selectedItems.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? t('common.loading') : t('restocking.placeOrder') }}
          </button>
        </div>

        <!-- Success Banner -->
        <div v-if="successMessage" class="success-banner">
          {{ successMessage }}
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()

    const budget = ref(50000)
    const recommendations = ref([])
    const manualOverrides = ref(new Set())
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const successMessage = ref('')

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const formatCurrency = (value) => {
      return value.toLocaleString('en-US', { minimumFractionDigits: 0, maximumFractionDigits: 0 })
    }

    // Greedy auto-selection based on current budget
    const autoSelected = computed(() => {
      const result = new Map()
      let remaining = budget.value
      for (const item of recommendations.value) {
        if (item.quantity_to_restock > 0 && item.line_total <= remaining) {
          result.set(item.id, true)
          remaining -= item.line_total
        } else {
          result.set(item.id, false)
        }
      }
      return result
    })

    // Apply manual overrides on top of auto-selection
    const effectiveSelected = (item) => {
      const auto = autoSelected.value.get(item.id) ?? false
      return manualOverrides.value.has(item.id) ? !auto : auto
    }

    const toggleItem = (item) => {
      if (item.quantity_to_restock === 0) return
      const overrides = new Set(manualOverrides.value)
      if (overrides.has(item.id)) {
        overrides.delete(item.id)
      } else {
        overrides.add(item.id)
      }
      manualOverrides.value = overrides
    }

    // Clear overrides when budget changes so auto-selection is fresh
    watch(budget, () => {
      manualOverrides.value = new Set()
    })

    const selectedItems = computed(() =>
      recommendations.value.filter(item => effectiveSelected(item) && item.quantity_to_restock > 0)
    )

    const budgetUsed = computed(() =>
      selectedItems.value.reduce((sum, item) => sum + item.line_total, 0)
    )

    const budgetRemaining = computed(() => budget.value - budgetUsed.value)

    // Determine which items are "over budget" — items that can't fit within the budget
    // given all previously selected items
    const isOverBudget = (item) => {
      if (item.quantity_to_restock === 0) return false
      if (effectiveSelected(item)) return false
      // Item is not selected; check if it could have been selected if budget allowed it
      // We consider it "over budget" if its line_total exceeds the remaining budget
      // after accounting for all selected items
      return item.line_total > budgetRemaining.value
    }

    const getRowClass = (item) => {
      if (item.quantity_to_restock === 0) return 'row-no-gap'
      if (effectiveSelected(item)) return 'row-selected'
      return 'row-unselected'
    }

    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        recommendations.value = await api.getRestockingRecommendations()
      } catch (err) {
        error.value = 'Failed to load restocking recommendations: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (selectedItems.value.length === 0 || submitting.value) return

      const items = recommendations.value
        .filter(item => effectiveSelected(item) && item.quantity_to_restock > 0)
        .map(item => ({
          sku: item.item_sku,
          name: item.item_name,
          quantity: item.quantity_to_restock,
          unit_price: item.unit_cost
        }))

      try {
        submitting.value = true
        const result = await api.createRestockingOrder(items)
        const orderNumber = result.order_number || result.id || 'RST-2025-0001'
        successMessage.value = t('restocking.orderPlaced', { number: orderNumber })
        manualOverrides.value = new Set()

        setTimeout(() => {
          successMessage.value = ''
        }, 5000)
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(() => {
      loadRecommendations()
    })

    return {
      t,
      budget,
      recommendations,
      loading,
      error,
      submitting,
      successMessage,
      currencySymbol,
      formatCurrency,
      autoSelected,
      effectiveSelected,
      toggleItem,
      selectedItems,
      budgetUsed,
      budgetRemaining,
      isOverBudget,
      getRowClass,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  /* inherits .main-content padding from App.vue */
}

/* Budget Card */
.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding-top: 0.5rem;
}

.slider-bound {
  font-size: 0.813rem;
  color: #64748b;
  font-weight: 500;
  white-space: nowrap;
  flex-shrink: 0;
}

.budget-slider {
  flex: 1;
  -webkit-appearance: none;
  appearance: none;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #fff;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  transition: box-shadow 0.15s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  box-shadow: 0 1px 8px rgba(37, 99, 235, 0.5);
}

.budget-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #fff;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.budget-slider::-webkit-slider-runnable-track {
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
}

.budget-slider::-moz-range-track {
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
}

/* Table */
.col-check {
  width: 40px;
  text-align: center;
}

.row-checkbox {
  width: 16px;
  height: 16px;
  cursor: pointer;
  accent-color: #2563eb;
}

.no-check-placeholder {
  display: inline-block;
  width: 16px;
  height: 16px;
}

.row-selected td {
  color: #0f172a;
}

.row-unselected td {
  color: #94a3b8;
}

.row-no-gap td {
  color: #94a3b8;
  background: #f8fafc;
}

.muted-text {
  color: #94a3b8;
  font-style: italic;
}

.line-total-cell {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.over-budget-badge {
  font-size: 0.688rem;
  padding: 0.188rem 0.5rem;
  white-space: nowrap;
}

/* No items state */
.no-items {
  padding: 3rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

/* Summary Bar */
.summary-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem 0 0;
  margin-top: 1rem;
  border-top: 1px solid #e2e8f0;
  gap: 1rem;
  flex-wrap: wrap;
}

.summary-stats {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  flex-wrap: wrap;
}

.summary-item {
  font-size: 0.875rem;
  color: #64748b;
}

.summary-item strong {
  color: #0f172a;
}

.summary-separator {
  color: #cbd5e1;
  font-size: 1rem;
}

.over-budget-text strong {
  color: #dc2626;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease, opacity 0.2s ease;
  white-space: nowrap;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Success Banner */
.success-banner {
  margin-top: 1rem;
  padding: 0.875rem 1rem;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 6px;
  color: #065f46;
  font-size: 0.875rem;
  font-weight: 500;
}
</style>
