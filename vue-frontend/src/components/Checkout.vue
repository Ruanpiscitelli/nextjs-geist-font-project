<template>
  <div class="max-w-md w-full bg-white p-6 rounded-lg shadow-md">
    <h2 class="text-2xl font-semibold mb-4 text-center">Checkout</h2>
    <form @submit.prevent="handleSubmit" class="space-y-4">
      <div>
        <label for="name" class="block text-sm font-medium text-gray-700">Name</label>
        <input v-model="form.name" id="name" type="text" required class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-black focus:ring-black" />
      </div>
      <div>
        <label for="email" class="block text-sm font-medium text-gray-700">Email</label>
        <input v-model="form.email" id="email" type="email" required class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-black focus:ring-black" />
      </div>
      <div>
        <label for="amount" class="block text-sm font-medium text-gray-700">Amount (USD cents)</label>
        <input v-model.number="form.amount" id="amount" type="number" min="1" required class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-black focus:ring-black" />
      </div>
      <div>
        <label class="block text-sm font-medium text-gray-700 mb-1">Payment Method</label>
        <select v-model="form.paymentMethod" required class="block w-full rounded-md border-gray-300 shadow-sm focus:border-black focus:ring-black">
          <option value="" disabled>Select a payment method</option>
          <option value="stripe">Stripe</option>
          <option value="mercadopago">Mercado Pago</option>
        </select>
      </div>
      <div v-if="form.paymentMethod === 'stripe'">
        <label class="block text-sm font-medium text-gray-700 mb-1">Card Details</label>
        <div id="card-element" class="p-2 border rounded-md"></div>
        <p v-if="cardError" class="text-red-600 text-sm mt-1">{{ cardError }}</p>
      </div>
      <button type="submit" :disabled="loading" class="w-full bg-black text-white py-2 rounded-md hover:bg-gray-900 transition">
        {{ loading ? 'Processing...' : 'Pay Now' }}
      </button>
    </form>
    <p v-if="errorMessage" class="text-red-600 mt-4 text-center">{{ errorMessage }}</p>
    <p v-if="successMessage" class="text-green-600 mt-4 text-center">{{ successMessage }}</p>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'
import { loadStripe } from '@stripe/stripe-js'

const form = ref({
  name: '',
  email: '',
  amount: 100,
  paymentMethod: '',
})

const loading = ref(false)
const errorMessage = ref('')
const successMessage = ref('')
const cardError = ref('')

let stripe = null
let cardElement = null

onMounted(async () => {
  if (form.value.paymentMethod === 'stripe') {
    await setupStripe()
  }
})

async function setupStripe() {
  stripe = await loadStripe('your_stripe_publishable_key_here')
  const elements = stripe.elements()
  cardElement = elements.create('card')
  cardElement.mount('#card-element')
  cardElement.on('change', (event) => {
    cardError.value = event.error ? event.error.message : ''
  })
}

async function handleSubmit() {
  errorMessage.value = ''
  successMessage.value = ''
  cardError.value = ''

  if (!form.value.paymentMethod) {
    errorMessage.value = 'Please select a payment method.'
    return
  }

  loading.value = true

  if (form.value.paymentMethod === 'stripe') {
    if (!stripe || !cardElement) {
      await setupStripe()
    }
    try {
      // Create payment intent on backend
      const response = await axios.post('http://localhost:8000/api/stripe/create-payment-intent', {
        amount: form.value.amount,
        currency: 'usd',
      })
      const clientSecret = response.data.clientSecret

      // Confirm card payment
      const result = await stripe.confirmCardPayment(clientSecret, {
        payment_method: {
          card: cardElement,
          billing_details: {
            name: form.value.name,
            email: form.value.email,
          },
        },
      })

      if (result.error) {
        errorMessage.value = result.error.message
      } else if (result.paymentIntent && result.paymentIntent.status === 'succeeded') {
        successMessage.value = 'Payment successful!'
      }
    } catch (error) {
      errorMessage.value = error.response?.data?.error || error.message
    }
  } else if (form.value.paymentMethod === 'mercadopago') {
    try {
      // Create preference on backend
      const response = await axios.post('http://localhost:8000/api/mercadopago/create-preference', {
        items: [
          {
            title: 'Order Payment',
            quantity: 1,
            unit_price: form.value.amount / 100,
          },
        ],
      })
      const initPoint = response.data.init_point
      window.location.href = initPoint
    } catch (error) {
      errorMessage.value = error.response?.data?.error || error.message
    }
  }

  loading.value = false
}
</script>

<style scoped>
#card-element {
  background-color: white;
}
</style>
</create_file>
