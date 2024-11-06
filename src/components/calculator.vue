<template>
    <div class="calculator-wrapper">
      <div class="calculator-container">
        <h1>Futures Income Calculator</h1>
  
        <div class="input-group">
          <div class="contract-type">
            <label class="switch">
              <input type="checkbox" v-model="isES">
              <span class="slider">
                <span class="option mes">MES</span>
                <span class="option es">ES</span>
              </span>
            </label>
          </div>
  
          <div class="input-field">
            <label>Number of Contracts</label>
            <input 
              type="number" 
              v-model="contracts" 
              min="1"
              placeholder="1"
            >
          </div>
  
          <div class="input-field">
            <label>Points Move</label>
            <input 
              type="number" 
              v-model="pointsMove" 
              step="0.25"
              placeholder="1"
            >
          </div>

          <div class="input-field">
            <label>Commission (Total)</label>
            <input 
              type="number" 
              v-model="commission" 
              step="0.01"
              placeholder="0"
            >
          </div>
        </div>
  
        <div class="currency-selector">
          <label>Secondary Currency:</label>
          <select v-model="selectedCurrency">
            <option 
              v-for="(rate, currency) in conversionRates" 
              :key="currency" 
              :value="currency"
            >
              {{ currency }} ({{ getCurrencySymbol(currency) }})
            </option>
          </select>
          <div class="last-updated" v-if="lastUpdated">
            Last updated: {{ lastUpdated }}
          </div>
        </div>
  
        <div class="results">
          <div class="result-item">
            <span>Daily P/L:</span>
            <div class="currency-values">
              <span>${{ dailyPL.toFixed(2) }}</span>
              <span>{{ currencySymbol }}{{ (dailyPL * conversionRate).toFixed(2) }}</span>
            </div>
          </div>
          <div class="result-item">
            <span>Monthly Income:</span>
            <div class="currency-values">
              <span>${{ monthlyIncome.toFixed(2) }}</span>
              <span>{{ currencySymbol }}{{ (monthlyIncome * conversionRate).toFixed(2) }}</span>
            </div>
          </div>
          <div class="result-item">
            <span>Annual Income:</span>
            <div class="currency-values">
              <span>${{ annualIncome.toFixed(2) }}</span>
              <span>{{ currencySymbol }}{{ (annualIncome * conversionRate).toFixed(2) }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </template>
  
  <script>
  export default {
    name: 'FuturesCalculator',
    data() {
      return {
        contracts: 1,
        pointsMove: 1,
        isES: false,
        selectedCurrency: 'MYR',
        conversionRates: {},
        isLoading: true,
        lastUpdated: null,
        commonSymbols: {
          USD: '$',
          EUR: '€',
          GBP: '£',
          JPY: '¥',
          AUD: 'A$',
          MYR: 'RM',
          // Add any other known symbols here
        },
        commission: 0,
      }
    },
    async created() {
      await this.fetchExchangeRates()
      // Update rates every 24 hours
      setInterval(this.fetchExchangeRates, 24 * 60 * 60 * 1000)
    },
    methods: {
      async fetchExchangeRates() {
        const API_KEY = import.meta.env.VITE_API_KEY
        
        if (!API_KEY) {
          console.error('API key not found. Please check your .env file')
          // Use fallback rates
          this.conversionRates = {
            EUR: 0.92,
            GBP: 0.79,
            JPY: 151.45,
            AUD: 1.53,
            MYR: 4.77
          }
          return
        }
  
        try {
          const response = await fetch(`https://v6.exchangerate-api.com/v6/${API_KEY}/latest/USD`, {
            method: 'GET',
            headers: {
              'Accept': 'application/json',
            }
          })
          
          if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`)
          }
          
          const data = await response.json()
          
          if (data.result === 'error') {
            throw new Error(data['error-type'])
          }
          
          // Store all conversion rates except USD (base currency)
          this.conversionRates = Object.fromEntries(
            Object.entries(data.conversion_rates)
              .filter(([currency]) => currency !== 'USD')
              .sort()
          )
          
          this.lastUpdated = new Date().toLocaleString()
          this.isLoading = false
          
          // If selected currency no longer exists in new rates, default to MYR
          if (!this.conversionRates[this.selectedCurrency]) {
            this.selectedCurrency = 'MYR'
          }
        } catch (error) {
          console.error('Error fetching exchange rates:', error)
          // Minimal fallback rates if API fails
          this.conversionRates = {
            EUR: 0.92,
            GBP: 0.79,
            JPY: 151.45,
            AUD: 1.53,
            MYR: 4.77
          }
        }
      },
      getCurrencySymbol(currency) {
        return this.commonSymbols[currency] || currency
      }
    },
    computed: {
      // ES is $50 per point, MES is $5 per point
      pointValue() {
        return this.isES ? 50 : 5
      },
      dailyPL() {
        return (this.contracts * this.pointsMove * this.pointValue) - this.commission
      },
      monthlyIncome() {
        return this.dailyPL * 21 // Average trading days in a month
      },
      annualIncome() {
        return this.monthlyIncome * 12
      },
      conversionRate() {
        return this.conversionRates[this.selectedCurrency]
      },
      currencySymbol() {
        return this.getCurrencySymbol(this.selectedCurrency)
      }
    }
  }
  </script>
  
  <style scoped>
  .calculator-wrapper {
    width: 100%;
    display: flex;
    justify-content: center;
    margin: 0;
    padding: 0;
  }
  
  .calculator-container {
    width: min(600px, 92vw);
    margin: 1rem auto;
    padding: 1rem;
    border-radius: 12px;
    background: white;
    color: black;
  }
  
  h1 {
    text-align: center;
    margin-bottom: 2rem;
  }
  
  .input-group {
    display: flex;
    flex-direction: column;
    gap: 1rem;
    margin-bottom: 1.5rem;
  }
  
  .input-field {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    text-align: center;
    color: black;
  }
  
  .input-field label {
    text-align: center;  /* Center the label text */
  }
  
  .input-field input {
    padding: 0.75rem;
    border: 1px solid #ddd;
    border-radius: 6px;
    text-align: center;
    font-size: 1rem;
    color: black;
  }
  
  .contract-type {
    display: flex;
    justify-content: center;
    align-items: center;
    margin: 0.5rem 0;
  }
  
  .switch {
    position: relative;
    display: inline-block;
    width: 200px;  /* Increased width */
    height: 50px;  /* Increased height */
  }
  
  .switch input {
    opacity: 0;
    width: 0;
    height: 0;
  }
  
  .slider {
    position: absolute;
    cursor: pointer;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: #ccc;
    transition: .4s;
    border-radius: 25px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0 15px;
  }
  
  .slider:before {
    position: absolute;
    content: "";
    height: 42px;  /* Increased height */
    width: 100px;  /* Increased width */
    left: 4px;
    bottom: 4px;
    background-color: white;
    transition: .4s;
    border-radius: 22px;
    z-index: 1;
  }
  
  input:checked + .slider {
    background-color: #ccc;
  }
  
  input:checked + .slider:before {
    transform: translateX(92px);  /* Adjusted translation */
  }
  
  .option {
    color: #666;
    font-weight: bold;
    z-index: 2;
    transition: .4s;
    text-transform: uppercase;
    width: 40%;
    text-align: center;
  }
  
  .mes {
    color: black;
  }
  
  input:checked + .slider .mes {
    color: #666;
  }
  
  input:checked + .slider .es {
    color: black;
  }
  
  .results {
    background: #f8f9fa;
    padding: 1.5rem;
    border-radius: 8px;
  }
  
  .result-item {
    display: flex;
    justify-content: space-between;
    margin-bottom: 0.5rem;
    font-size: 1.1rem;
    color: black;
  }
  
  .currency-values {
    display: flex;
    gap: 1.5rem;  /* Space between USD and EUR values */
    font-weight: bold;
  }
  
  .currency-values span {
    min-width: 120px;  /* Ensures consistent width for numbers */
    text-align: right;
  }
  
  .result-item:last-child {
    margin-bottom: 0;
  }
  
  .currency-selector {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    margin-bottom: 1.5rem;
    max-width: 200px;
    margin-left: auto;
    text-align: center;
  }
  
  .currency-selector select {
    padding: 0.5rem;
    border: 1px solid #ddd;
    border-radius: 6px;
    font-size: 1rem;
    color: black;
    background-color: rgb(244, 244, 244);
    width: auto;
    min-width: 120px;
    text-align: center;
  }
  
  .last-updated {
    font-size: 0.8rem;
    color: #666;
    text-align: right;
    width: 100%;
    white-space: nowrap;
    margin-bottom: 1rem;
  }
  
  /* Media query for landscape/wider screens */
  @media (min-width: 768px) {
    .calculator-container {
      padding: 2rem;
    }
    
    .input-group {
      gap: 1.5rem;
    }
  }
  
  /* Media query for smaller screens */
  @media (max-width: 480px) {
    .calculator-wrapper {
      padding: 0;
      margin: 0;
      min-width: 100%;
      overflow-x: hidden;
      min-height: 100vh;  /* Added to help with vertical centering */
      align-items: center;  /* Added to center vertically */
    }
  
    .calculator-container {
      width: 90vw;
      margin: 0rem auto;  /* Increased top/bottom margin from 0.5rem to 2rem */
      margin-top: -4rem;
      padding: 0.5rem;
    }
  
    .currency-values {
      gap: 0.5rem;
    }
  
    .currency-values span {
      min-width: 65px;
      font-size: 1.1rem;
    }
  
    .results {
      padding: 0.75rem;
    }
  
    .result-item {
      font-size: .95rem;
    }
  
    /* Make currency selector more compact */
    .currency-selector {
      gap: 0.5rem;
      font-size: 0.85rem;
      margin: 0 auto;  /* Center the entire selector */
      text-align: center;  /* Center the label text */
    }
  
    .currency-selector select {
      min-width: auto;
      max-width: 100px;
      margin: 0 auto;  /* Center the dropdown */
    }
  
    .last-updated {
      font-size: 0.7rem;
      text-align: center;  /* Center the last updated text */
    }
  
    .input-field input {
      width: 60%;  /* Make input fields smaller on mobile */
      margin: 0 auto;  /* Center the input box */
    }
  }
  
  /* Extra small screens */
  @media (max-width: 360px) {
    .calculator-container {
      width: 88vw;
      padding: 0.4rem;
    }
  
    .currency-values span {
      min-width: 60px;
    }
  }
  
  /* Remove the transform that was pushing content down */
  .calculator-container {
    transform: none;
  }
  </style>
  
  