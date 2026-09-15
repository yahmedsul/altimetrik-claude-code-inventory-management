<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>AI-recommended items to restock based on demand forecasts and current inventory levels.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- ── Section 1: Budget Input ──────────────────────────────────── -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
        </div>
        <div class="budget-input-row">
          <div class="input-group">
            <span class="currency-prefix">{{ currencySymbol }}</span>
            <input
              type="number"
              v-model.number="budget"
              min="0"
              class="budget-input"
              placeholder="50000"
            />
          </div>
        </div>
        <div class="budget-summary">
          Recommended items: {{ recommendedItems.length }}
          &nbsp;&middot;&nbsp;
          Estimated cost: {{ formatCurrency(totalCost) }}
          &nbsp;&middot;&nbsp;
          Remaining: {{ formatCurrency(remainingBudget) }}
        </div>
      </div>

      <!-- ── Success / Error banners ───────────────────────────────────── -->
      <div v-if="successMessage" class="banner success-banner">
        {{ successMessage }}
      </div>
      <div v-if="submitError" class="banner error-banner">
        {{ submitError }}
      </div>

      <!-- ── Section 2: Recommended Items Table ────────────────────────── -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Restocking</h3>
        </div>

        <div v-if="recommendedItems.length === 0" class="empty-state">
          No items fit within the current budget. Try increasing your budget.
        </div>

        <div v-else>
          <div class="table-container">
            <table>
              <thead>
                <tr>
                  <th>Item</th>
                  <th>SKU</th>
                  <th>Category</th>
                  <th>Warehouse</th>
                  <th>On Hand</th>
                  <th>Reorder Point</th>
                  <th>Restock Qty</th>
                  <th>Unit Cost</th>
                  <th>Line Total</th>
                  <th>Trend</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="item in recommendedItems" :key="item.sku">
                  <td>{{ item.name }}</td>
                  <td><strong>{{ item.sku }}</strong></td>
                  <td>{{ item.category }}</td>
                  <td>{{ item.warehouse }}</td>
                  <td>{{ item.quantity_on_hand }}</td>
                  <td>{{ item.reorder_point }}</td>
                  <td><strong>{{ item.restock_qty }}</strong></td>
                  <td>{{ formatCurrency(item.unit_cost) }}</td>
                  <td><strong>{{ formatCurrency(item.line_total) }}</strong></td>
                  <td>
                    <!-- Local trend-badge classes keep this view's gray-for-decreasing
                         independent of the global .badge.decreasing (red) style -->
                    <span :class="['trend-badge', item.trend]">{{ item.trend }}</span>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
          <div class="table-summary">
            Total: {{ formatCurrency(totalCost) }} across {{ recommendedItems.length }}
            {{ recommendedItems.length === 1 ? 'item' : 'items' }}
          </div>
        </div>
      </div>

      <!-- ── Section 3: Place Order ─────────────────────────────────────── -->
      <div class="place-order-section">
        <button
          class="btn-primary"
          :disabled="recommendedItems.length === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? 'Submitting...' : 'Place Order' }}
        </button>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

// JPY conversion rate applied to all USD-denominated values from the API
const JPY_RATE = 150

// Trend sort priority: lower number = higher priority in the greedy selection pass
const TREND_PRIORITY = { increasing: 0, stable: 1, decreasing: 2 }

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()

    // ── State ──────────────────────────────────────────────────────────
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const successMessage = ref(null)
    const submitError = ref(null)

    // User-controlled budget (default $50,000). Input value is always in the
    // display currency so that the label and prefix stay consistent.
    const budget = ref(50000)

    // Raw data from API — all items, unfiltered
    const demandForecasts = ref([])
    const inventoryItems = ref([])

    // ── Currency helpers ───────────────────────────────────────────────
    const currencySymbol = computed(() =>
      currentCurrency.value === 'JPY' ? '¥' : '$'
    )

    /**
     * Formats a USD-denominated value for display.
     * When the locale is JPY, converts at JPY_RATE and uses ¥ symbol.
     */
    const formatCurrency = (usdValue) => {
      if (currentCurrency.value === 'JPY') {
        const jpy = Math.round(usdValue * JPY_RATE)
        return '¥' + jpy.toLocaleString()
      }
      return (
        '$' +
        Number(usdValue).toLocaleString('en-US', {
          minimumFractionDigits: 2,
          maximumFractionDigits: 2
        })
      )
    }

    /**
     * Converts the user's budget input (display currency) back to USD so it
     * can be compared against line_total values (which are always in USD).
     */
    const budgetInUsd = computed(() =>
      currentCurrency.value === 'JPY' ? budget.value / JPY_RATE : budget.value
    )

    // ── Recommendation algorithm ───────────────────────────────────────
    /**
     * Greedy, budget-constrained restocking recommendations.
     *
     * Steps:
     *  1. Build an inventory lookup map keyed by SKU.
     *  2. For each demand forecast find the matching inventory record.
     *  3. Compute restock_qty = max(0, reorder_point - quantity_on_hand).
     *     Skip items that don't need restocking (restock_qty === 0).
     *  4. Compute line_total = restock_qty × unit_cost.
     *  5. Sort candidates by trend priority: increasing → stable → decreasing.
     *  6. Walk the sorted list and include each item only if its full
     *     line_total fits within the remaining budget (greedy knapsack).
     */
    const recommendedItems = computed(() => {
      // Step 1 — build a fast SKU → inventory lookup
      const inventoryBySku = {}
      for (const inv of inventoryItems.value) {
        inventoryBySku[inv.sku] = inv
      }

      // Steps 2–4 — build the candidate list
      const candidates = []
      for (const forecast of demandForecasts.value) {
        const inv = inventoryBySku[forecast.item_sku]
        if (!inv) continue // No matching inventory record for this SKU

        const restock_qty = Math.max(0, inv.reorder_point - inv.quantity_on_hand)
        if (restock_qty <= 0) continue // Item is adequately stocked; skip

        const line_total = restock_qty * inv.unit_cost

        candidates.push({
          // Inventory fields used in table and submit payload
          sku: inv.sku,
          name: inv.name,
          category: inv.category,
          warehouse: inv.warehouse,
          quantity_on_hand: inv.quantity_on_hand,
          reorder_point: inv.reorder_point,
          unit_cost: inv.unit_cost,
          // Derived restocking values
          restock_qty,
          line_total,
          // Demand signal used for priority sorting
          trend: forecast.trend
        })
      }

      // Step 5 — sort by trend priority (ascending = highest priority first)
      candidates.sort((a, b) => {
        const pa = TREND_PRIORITY[a.trend] ?? 3
        const pb = TREND_PRIORITY[b.trend] ?? 3
        return pa - pb
      })

      // Step 6 — greedy selection: include item only if it fits within remaining budget
      let remaining = budgetInUsd.value
      const selected = []
      for (const item of candidates) {
        if (item.line_total <= remaining) {
          selected.push(item)
          remaining -= item.line_total
        }
      }

      return selected
    })

    // Total cost of all recommended items (USD)
    const totalCost = computed(() =>
      recommendedItems.value.reduce((sum, item) => sum + item.line_total, 0)
    )

    // Budget remaining after covering the recommended items (USD for internal math;
    // formatCurrency() handles display conversion)
    const remainingBudget = computed(() => budgetInUsd.value - totalCost.value)

    // ── Data loading ───────────────────────────────────────────────────
    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        // Fetch demand and full (unfiltered) inventory in parallel
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory() // No filters — need all items for SKU matching
        ])
        demandForecasts.value = forecasts
        inventoryItems.value = inventory
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // ── Order submission ───────────────────────────────────────────────
    /**
     * Generates a fallback order ID in the format RST-XXXXXXXX when the
     * backend response does not include one.
     */
    const generateOrderId = () => {
      const hex = Math.floor(Math.random() * 0xffffffff)
        .toString(16)
        .toUpperCase()
        .padStart(8, '0')
      return `RST-${hex}`
    }

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0 || submitting.value) return

      submitting.value = true
      submitError.value = null
      successMessage.value = null

      // Build payload matching backend CreateRestockingOrderRequest schema
      const payload = {
        items: recommendedItems.value.map((item) => ({
          sku: item.sku,
          name: item.name,
          category: item.category,
          warehouse: item.warehouse,
          quantity: item.restock_qty,
          unit_cost: item.unit_cost,
          line_total: item.line_total,
          trend: item.trend
        })),
        total_cost: totalCost.value,
        order_date: new Date().toISOString().slice(0, 10) // YYYY-MM-DD
      }

      try {
        const response = await api.submitRestockingOrder(payload)

        // Use backend-provided order ID when available, otherwise generate one
        const orderId =
          response?.order_id || response?.id || generateOrderId()

        // Expected delivery = 7 days from today
        const delivery = new Date()
        delivery.setDate(delivery.getDate() + 7)
        const deliveryStr = delivery.toLocaleDateString('en-US', {
          month: 'short',
          day: 'numeric',
          year: 'numeric'
        })

        successMessage.value = `Order ${orderId}-001 submitted successfully! Expected delivery: ${deliveryStr}`

        // Auto-dismiss the success banner after 5 seconds
        setTimeout(() => {
          successMessage.value = null
        }, 5000)
      } catch (err) {
        submitError.value =
          'Failed to submit order: ' +
          (err.response?.data?.detail || err.message)
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      loading,
      error,
      budget,
      submitting,
      successMessage,
      submitError,
      currencySymbol,
      recommendedItems,
      totalCost,
      remainingBudget,
      formatCurrency,
      placeOrder
    }
  }
}
</script>

<style scoped>
/* ── Budget card ─────────────────────────────────────────────────────── */
.budget-card {
  max-width: 560px;
}

.budget-input-row {
  margin-bottom: 0.875rem;
}

.input-group {
  display: inline-flex;
  align-items: center;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  overflow: hidden;
  background: #ffffff;
  transition: border-color 0.2s, box-shadow 0.2s;
}

.input-group:focus-within {
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
}

.currency-prefix {
  padding: 0.625rem 0.875rem;
  background: #f8fafc;
  border-right: 1px solid #e2e8f0;
  color: #475569;
  font-weight: 600;
  font-size: 1rem;
  user-select: none;
}

.budget-input {
  border: none;
  outline: none;
  padding: 0.625rem 1rem;
  font-size: 1rem;
  color: #0f172a;
  width: 220px;
  background: transparent;
  font-family: inherit;
}

/* Remove native number-input spinner arrows */
.budget-input::-webkit-outer-spin-button,
.budget-input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}
.budget-input[type='number'] {
  -moz-appearance: textfield;
}

.budget-summary {
  font-size: 0.875rem;
  color: #64748b;
}

/* ── Banners ─────────────────────────────────────────────────────────── */
.banner {
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 500;
  margin-bottom: 1.25rem;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #a7f3d0;
  color: #065f46;
}

.error-banner {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
}

/* ── Empty state ─────────────────────────────────────────────────────── */
.empty-state {
  padding: 3rem 1.5rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

/* ── Trend badges (local — overrides global .badge.decreasing red) ───── */
.trend-badge {
  display: inline-block;
  padding: 0.313rem 0.75rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.trend-badge.increasing {
  background: #d1fae5;
  color: #065f46;
}

.trend-badge.stable {
  background: #dbeafe;
  color: #1e40af;
}

/* Gray badge for decreasing (task spec). The global .badge.decreasing is red;
   using a separate local class avoids any conflict with that global rule. */
.trend-badge.decreasing {
  background: #f1f5f9;
  color: #64748b;
}

/* ── Table summary row ───────────────────────────────────────────────── */
.table-summary {
  padding: 0.75rem;
  border-top: 2px solid #e2e8f0;
  font-size: 0.875rem;
  font-weight: 600;
  color: #0f172a;
  text-align: right;
  background: #f8fafc;
  border-radius: 0 0 8px 8px;
}

/* ── Place Order section ─────────────────────────────────────────────── */
.place-order-section {
  display: flex;
  justify-content: flex-end;
  margin-bottom: 2rem;
}

.btn-primary {
  background: #2563eb;
  color: #ffffff;
  border: none;
  border-radius: 8px;
  padding: 0.75rem 2rem;
  font-size: 0.938rem;
  font-weight: 600;
  font-family: inherit;
  cursor: pointer;
  transition: background 0.2s, opacity 0.2s;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
