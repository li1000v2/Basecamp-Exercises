<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Recommend items to restock from demand forecasts, within your budget.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Control Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Budget &amp; Delivery</h3>
        </div>
        <div class="budget-grid">
          <div class="budget-slider-area">
            <div class="stat-label">Budget</div>
            <div class="budget-display">{{ formatMoney(budget) }}</div>
            <input
              type="range"
              class="budget-slider"
              min="0"
              max="500000"
              step="1000"
              v-model.number="budget"
            />
            <div class="slider-range-labels">
              <span>{{ formatMoney(0) }}</span>
              <span>{{ formatMoney(500000) }}</span>
            </div>
          </div>
          <div class="tier-area">
            <div class="stat-label">Delivery Tier</div>
            <select class="tier-select" v-model="deliveryTier">
              <option value="Express">Express (7 days)</option>
              <option value="Standard">Standard (14 days)</option>
              <option value="Economy">Economy (30 days)</option>
            </select>
          </div>
        </div>
      </div>

      <!-- Recommendations Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items</h3>
          <span class="card-count">({{ recommendations.length }} of {{ totalCandidates }} shown within budget)</span>
        </div>

        <div v-if="recommendations.length === 0" class="empty-state">
          No items currently match the shortfall criteria within this budget. Try increasing the budget, or check that demand forecasts reflect current inventory levels.
        </div>
        <div v-else>
          <div class="table-container">
            <table>
              <thead>
                <tr>
                  <th>SKU</th>
                  <th>Item Name</th>
                  <th>Trend</th>
                  <th>Current Stock</th>
                  <th>Forecasted Demand</th>
                  <th>Recommended Qty</th>
                  <th>Unit Cost</th>
                  <th>Item Total</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="row in recommendations" :key="row.sku">
                  <td><strong>{{ row.sku }}</strong></td>
                  <td>{{ row.name }}</td>
                  <td>
                    <span :class="['badge', row.trend]">{{ capitalize(row.trend) }}</span>
                  </td>
                  <td>{{ row.current_stock.toLocaleString() }}</td>
                  <td>{{ row.forecasted_demand.toLocaleString() }}</td>
                  <td>{{ row.recommended_qty.toLocaleString() }}</td>
                  <td>{{ formatMoney(row.unit_cost) }}</td>
                  <td><strong>{{ formatMoney(row.item_total) }}</strong></td>
                </tr>
              </tbody>
            </table>
          </div>

          <div class="summary-row">
            <div class="summary-item">
              <span class="summary-label">Items</span>
              <span class="summary-value">{{ recommendations.length }}</span>
            </div>
            <div class="summary-item">
              <span class="summary-label">Total</span>
              <span class="summary-value">{{ formatMoney(totalCost) }}</span>
            </div>
            <div class="summary-item">
              <span class="summary-label">Budget</span>
              <span class="summary-value">{{ formatMoney(budget) }}</span>
            </div>
            <div class="summary-item">
              <span class="summary-label">Remaining</span>
              <span class="summary-value" :class="remainingBudget < 0 ? 'value-negative' : 'value-positive'">
                {{ formatMoney(remainingBudget) }}
              </span>
            </div>
          </div>
        </div>

        <div class="action-row">
          <button
            class="place-order-btn"
            :disabled="recommendations.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Submitting...' : 'Place Order' }}
          </button>

          <div v-if="submitSuccess" class="success-message">
            Order {{ submitSuccess.orderNumber }} submitted.
            <router-link to="/orders">View in Orders &rarr;</router-link>
          </div>

          <div v-if="submitError" class="error">{{ submitError }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'
import { formatCurrencyWithDecimals } from '../utils/currency'

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()

    const budget = ref(50000)
    const deliveryTier = ref('Standard')
    const demandForecasts = ref([])
    const inventoryItems = ref([])
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const submitSuccess = ref(null)
    const submitError = ref(null)

    const TREND_RANK = { increasing: 0, stable: 1, decreasing: 2 }

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        demandForecasts.value = forecasts
        inventoryItems.value = inventory
      } catch (err) {
        error.value = 'Failed to load forecast/inventory data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const sortedCandidates = computed(() => {
      const invBySku = new Map(inventoryItems.value.map(i => [i.sku, i]))
      const rows = []
      for (const f of demandForecasts.value) {
        const inv = invBySku.get(f.item_sku)
        if (!inv) continue
        if (f.forecasted_demand <= inv.quantity_on_hand) continue
        const gap = f.forecasted_demand - inv.quantity_on_hand
        const recommended_qty = Math.ceil(gap / 10) * 10
        const item_total = recommended_qty * inv.unit_cost
        rows.push({
          sku: f.item_sku,
          name: f.item_name,
          trend: f.trend,
          current_stock: inv.quantity_on_hand,
          forecasted_demand: f.forecasted_demand,
          recommended_qty,
          unit_cost: inv.unit_cost,
          item_total,
          gap
        })
      }
      rows.sort((a, b) => {
        const ta = TREND_RANK[a.trend] ?? 99
        const tb = TREND_RANK[b.trend] ?? 99
        if (ta !== tb) return ta - tb
        return b.gap - a.gap
      })
      return rows
    })

    const recommendations = computed(() => {
      const picked = []
      let remaining = budget.value
      for (const row of sortedCandidates.value) {
        if (row.item_total <= remaining) {
          picked.push(row)
          remaining -= row.item_total
        }
      }
      return picked
    })

    const totalCost = computed(() => recommendations.value.reduce((s, r) => s + r.item_total, 0))
    const remainingBudget = computed(() => budget.value - totalCost.value)
    const totalCandidates = computed(() => sortedCandidates.value.length)

    const placeOrder = async () => {
      submitError.value = null
      submitSuccess.value = null
      submitting.value = true
      try {
        const payload = {
          items: recommendations.value.map(r => ({
            sku: r.sku,
            name: r.name,
            quantity: r.recommended_qty,
            unit_price: r.unit_cost
          })),
          delivery_tier: deliveryTier.value
        }
        const created = await api.submitRestockingOrder(payload)
        submitSuccess.value = { orderNumber: created.order_number }
      } catch (err) {
        submitError.value = 'Failed to submit order: ' + (err.response?.data?.detail || err.message)
      } finally {
        submitting.value = false
      }
    }

    const formatMoney = (n) => formatCurrencyWithDecimals(n, currentCurrency.value, 0)

    const capitalize = (str) => str ? str.charAt(0).toUpperCase() + str.slice(1) : ''

    onMounted(loadData)

    return {
      budget,
      deliveryTier,
      loading,
      error,
      submitting,
      submitSuccess,
      submitError,
      recommendations,
      totalCost,
      remainingBudget,
      totalCandidates,
      placeOrder,
      formatMoney,
      capitalize,
      currentCurrency
    }
  }
}
</script>

<style scoped>
.budget-grid {
  display: flex;
  gap: 2rem;
  align-items: flex-start;
}

.budget-slider-area {
  flex: 1;
  min-width: 0;
}

.budget-display {
  font-size: 2.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  margin-bottom: 0.75rem;
  margin-top: 0.375rem;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  height: 6px;
  cursor: pointer;
  margin-bottom: 0.375rem;
}

.slider-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #64748b;
}

.tier-area {
  flex: 0 0 220px;
}

.tier-select {
  width: 100%;
  border: 1px solid #e2e8f0;
  padding: 0.625rem 1rem;
  border-radius: 6px;
  font-size: 0.875rem;
  color: #0f172a;
  background: white;
  cursor: pointer;
  outline: none;
  margin-top: 0.375rem;
  transition: border-color 0.2s;
}

.tier-select:focus {
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
}

.card-count {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 400;
}

.summary-row {
  display: flex;
  gap: 2rem;
  padding: 1rem 0.75rem;
  border-top: 1px solid #e2e8f0;
  margin-top: 0.5rem;
  font-weight: 500;
  flex-wrap: wrap;
}

.summary-item {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.summary-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.summary-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.value-positive {
  color: #059669;
}

.value-negative {
  color: #dc2626;
}

.action-row {
  padding: 1rem 0 0.25rem;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  align-items: flex-start;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  padding: 0.75rem 2rem;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.938rem;
  border: none;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.success-message {
  background: #f0fdf4;
  border-left: 4px solid #22c55e;
  color: #166534;
  padding: 0.75rem 1rem;
  border-radius: 0 6px 6px 0;
  font-size: 0.938rem;
  font-weight: 500;
}

.success-message a {
  color: #15803d;
  font-weight: 600;
  text-decoration: underline;
  margin-left: 0.375rem;
}

.success-message a:hover {
  color: #166534;
}

.empty-state {
  color: #64748b;
  padding: 2rem;
  text-align: center;
  font-size: 0.938rem;
}
</style>
