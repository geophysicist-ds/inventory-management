<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Budget-based restocking recommendations from demand forecasts</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div v-if="submittedOrder" class="success-banner">
        <div>
          <strong>Order {{ submittedOrder.order_number }} placed successfully.</strong>
          Expected delivery: {{ formatDeliveryDate(submittedOrder.expected_delivery) }}
        </div>
        <router-link to="/orders" class="view-orders-link">View in Orders</router-link>
      </div>

      <div v-if="submitError" class="error" style="margin-bottom: 1.25rem;">{{ submitError }}</div>

      <div class="card" style="margin-bottom: 1.5rem;">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
        </div>
        <div style="padding: 1.25rem 1.5rem;">
          <div class="budget-control">
            <input
              type="range"
              class="budget-slider"
              min="0"
              max="100000"
              step="1000"
              v-model.number="budget"
            />
            <input
              type="number"
              class="budget-number-input"
              min="0"
              max="100000"
              step="1000"
              v-model.number="budget"
            />
            <div class="budget-value">{{ formatCurrency(budget) }}</div>
          </div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items</h3>
          <button
            class="btn-primary"
            :disabled="recommendedItems.length === 0 || submitting || submitted"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>

        <div v-if="suggestions.length === 0" style="padding: 2rem 1.5rem; color: #64748b;">
          No restocking recommendations at this time.
        </div>
        <div v-else-if="recommendedItems.length === 0" style="padding: 2rem 1.5rem; color: #64748b;">
          No items fit within the current budget. Increase the budget to see recommendations.
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Forecasted Demand</th>
                <th>Qty to Order</th>
                <th>Unit Cost</th>
                <th>Line Total</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.item_sku">
                <td>{{ item.item_name }}</td>
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.forecasted_demand.toLocaleString() }}</td>
                <td>{{ item.recommended_quantity.toLocaleString() }}</td>
                <td>{{ formatCurrency(item.unit_cost) }}</td>
                <td><strong>{{ formatCurrency(item.unit_cost * item.recommended_quantity) }}</strong></td>
              </tr>
            </tbody>
            <tfoot>
              <tr class="summary-row">
                <td colspan="5">Total Cost</td>
                <td>{{ formatCurrency(totalCost) }}</td>
              </tr>
              <tr class="summary-row">
                <td colspan="5">Remaining Budget</td>
                <td>{{ formatCurrency(remainingBudget) }}</td>
              </tr>
            </tfoot>
          </table>
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
    const suggestions = ref([])
    const loading = ref(true)
    const error = ref(null)
    const submitError = ref(null)

    const budget = ref(10000)
    const submitting = ref(false)
    const submitted = ref(false)
    const submittedOrder = ref(null)

    const recommendedItems = computed(() => {
      let remainingBudget = budget.value
      const included = []

      for (const item of suggestions.value) {
        const lineCost = item.unit_cost * item.recommended_quantity
        if (lineCost <= remainingBudget) {
          included.push(item)
          remainingBudget -= lineCost
        }
      }

      return included
    })

    const totalCost = computed(() => {
      return recommendedItems.value.reduce(
        (sum, item) => sum + item.unit_cost * item.recommended_quantity,
        0
      )
    })

    const remainingBudget = computed(() => {
      return budget.value - totalCost.value
    })

    const loadSuggestions = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRestockingSuggestions()
        suggestions.value = data
      } catch (err) {
        error.value = 'Failed to load restocking suggestions: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitting.value = true
      submitError.value = null
      try {
        const orderPayload = {
          items: recommendedItems.value.map(item => ({
            sku: item.item_sku,
            name: item.item_name,
            quantity: item.recommended_quantity,
            unit_price: item.unit_cost
          })),
          warehouse: 'San Francisco'
        }
        const response = await api.submitRestockingOrder(orderPayload)
        submittedOrder.value = response
        submitted.value = true
      } catch (err) {
        submitError.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    const formatCurrency = (value) => {
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD' })
    }

    const formatDeliveryDate = (dateStr) => {
      const date = new Date(dateStr)
      if (isNaN(date.getTime())) return dateStr
      return date.toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric' })
    }

    onMounted(loadSuggestions)

    return {
      suggestions,
      loading,
      error,
      submitError,
      budget,
      submitting,
      submitted,
      submittedOrder,
      recommendedItems,
      totalCost,
      remainingBudget,
      placeOrder,
      formatCurrency,
      formatDeliveryDate
    }
  }
}
</script>

<style scoped>
.budget-control {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-bottom: 1.5rem;
}

.budget-slider {
  flex: 1;
}

.budget-number-input {
  width: 110px;
  padding: 0.375rem 0.625rem;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.938rem;
  color: #0f172a;
}

.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  min-width: 120px;
  text-align: right;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.view-orders-link {
  color: #065f46;
  font-weight: 600;
  text-decoration: underline;
  white-space: nowrap;
  margin-left: 1rem;
}

.view-orders-link:hover {
  color: #047857;
}

.summary-row td {
  font-weight: 600;
  background: #f8fafc;
  border-top: 2px solid #e2e8f0;
}

.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-weight: 600;
  font-size: 0.938rem;
  cursor: pointer;
  transition: background 0.2s;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
