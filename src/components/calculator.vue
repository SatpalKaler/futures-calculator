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
            <select v-model="contracts" class="points-move">
              <option v-for="n in 50" :key="n" :value="n">{{ n }}</option>
            </select>
          </div>
  
          <div class="input-field">
            <label>Points Move</label>
            <select v-model="pointsMove" class="points-move">
              <option 
                v-for="n in 2000" 
                :key="n" 
                :value="n * 0.25"
              >
                {{ (n * 0.25).toFixed(2) }}
              </option>
            </select>
          </div>

          <div class="input-row">
            <div class="input-field half-width">
              <label title="Commission">Commission (total)</label>
              <input 
                type="number" 
                v-model="commission" 
                step="0.01"
                placeholder="1.24"
              >
            </div>
            <div class="input-field half-width">
              <label>Win Rate</label>
              <select v-model="winRate">
                <option 
                  v-for="n in 13" 
                  :key="n" 
                  :value="(n * 5 + 35)"
                >
                  {{ n * 5 + 35 }}%
                </option>
              </select>
            </div>
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
            <div class="result-label">Daily P/L</div>
            <div class="currency-values">
              <span class="primary-currency">${{ formatNumber(dailyPL) }}</span>
              <span class="separator">|</span>
              <span class="secondary-currency">{{ currencySymbol }}{{ formatNumber(dailyPL * conversionRate) }}</span>
            </div>
          </div>
          <div class="result-item">
            <div class="result-label">Monthly Income</div>
            <div class="currency-values">
              <span class="primary-currency">${{ formatNumber(monthlyIncome) }}</span>
              <span class="separator">|</span>
              <span class="secondary-currency">{{ currencySymbol }}{{ formatNumber(monthlyIncome * conversionRate) }}</span>
            </div>
          </div>
          <div class="result-item">
            <div class="result-label">Annual Income</div>
            <div class="currency-values">
              <span class="primary-currency">${{ formatNumber(annualIncome) }}</span>
              <span class="separator">|</span>
              <span class="secondary-currency">{{ currencySymbol }}{{ formatNumber(annualIncome * conversionRate) }}</span>
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
        commission: 1.24,
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
        winRate: 60,  // default to 60%
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
      },
      formatNumber(value) {
        return new Intl.NumberFormat('en-US', {
          minimumFractionDigits: 2,
          maximumFractionDigits: 2
        }).format(value);
      },
    },
    computed: {
      // ES is $50 per point, MES is $5 per point
      pointValue() {
        return this.isES ? 50 : 5
      },
      dailyPL() {
        const grossPL = this.contracts * this.pointsMove * this.pointValue;
        const winAmount = grossPL * (this.winRate / 100);
        const lossAmount = grossPL * ((100 - this.winRate) / 100) * -1;
        return (winAmount + lossAmount) - this.commission;
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
    width: min(800px, 96vw);
    margin: 0.25rem auto;
    padding: 0.5rem;
    border-radius: 12px;
    background: white;
    color: black;
  }
  
  h1 {
    text-align: center;
    margin-bottom: 0.75rem;
    font-size: 1.5rem;
  }
  
  .input-group {
    display: flex;
    flex-direction: column;
    gap: 0.35rem;
    margin-bottom: 0.75rem;
  }
  
  .input-field {
    display: flex;
    flex-direction: column;
    gap: 0.15rem;
    text-align: center;
    color: black;
    position: relative;
  }
  
  .input-field label {
    text-align: center;  /* Center the label text */
    font-size: 0.9rem;
  }
  
  .input-field input,
  .input-field select {
    padding: 0.4rem;
    border: 1px solid #ddd;
    border-radius: 6px;
    text-align: center;
    font-size: 0.95rem;
    color: black;
    width: 50%;
    text-align-last: center;  /* This helps center text in dropdowns for some browsers */
    -moz-text-align-last: center;  /* Firefox support */
    -webkit-text-align-last: center;  /* Safari support */
  }
  
  /* Ensure options within select are centered */
  .input-field select option {
    text-align: center;
  }
  
  /* Ensure number inputs are centered */
  .input-field input[type="number"] {
    -moz-appearance: textfield;  /* Firefox */
    text-align: center;
  }
  
  /* Remove spinner buttons from number inputs while maintaining center alignment */
  .input-field input[type="number"]::-webkit-inner-spin-button,
  .input-field input[type="number"]::-webkit-outer-spin-button {
    -webkit-appearance: none;
    margin: 0;
  }
  
  .contract-type {
    display: flex;
    justify-content: center;
    align-items: center;
    margin: 0.25rem 0;
  }
  
  .switch {
    position: relative;
    display: inline-block;
    width: 160px;
    height: 40px;
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
    height: 34px;
    width: 80px;
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
    transform: translateX(72px);
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
    padding: 0.75rem;
    border-radius: 8px;
  }
  
  .result-item {
    display: flex;
    flex-direction: column;
    margin-bottom: 0.75rem;
    text-align: center;
  }
  
  .result-label {
    font-weight: 500;
    margin-bottom: 0.25rem;
    font-size: 0.8rem;
    color: #666;
  }
  
  .currency-values {
    display: flex;
    justify-content: center;
    align-items: center;
    gap: 0.5rem;
    font-size: 1.2rem;
  }
  
  .primary-currency {
    font-weight: bold;
  }
  
  .secondary-currency {
    color: #666;
    font-weight: normal;
  }
  
  .separator {
    color: #ddd;
    font-weight: 300;
    padding: 0 0.25rem;
  }
  
  .result-item:last-child {
    margin-bottom: 0;
  }
  
  .currency-selector {
    display: flex;
    flex-direction: column;
    gap: 0.15rem;
    margin-bottom: 0.75rem;
    max-width: 200px;
    margin: 0 auto;  /* Center the currency selector */
    text-align: center;
  }
  
  .currency-selector select {
    padding: 0.4rem;
    border: 1px solid #ddd;
    border-radius: 6px;
    font-size: 0.95rem;
    color: black;
    background-color: rgb(244, 244, 244);
    width: auto;
    min-width: 120px;
    text-align: center;
    margin: 0 auto;  /* Center the dropdown */
    display: block;  /* Ensures margin auto works */
  }
  
  .last-updated {
    font-size: 0.75rem;
    color: #666;
    text-align: center;  /* Center the last updated text */
    width: 100%;
    white-space: nowrap;
    margin-bottom: 0.35rem;
  }
  
  /* Media query for landscape/wider screens */
  @media (min-width: 768px) {
    .calculator-container {
      padding: 1rem;
    }
    
    .input-group {
      gap: 0.75rem;
    }
  }
  
  /* Media query for smaller screens */
  @media (max-width: 480px) {
    .calculator-wrapper {
      padding: 0;
      margin: 0;
      min-width: 100%;
      overflow-x: hidden;
      min-height: 100dvh;
      align-items: center;
    }
  
    .calculator-container {
      width: 96vw;
      margin: 0 auto;
      padding: 0.5rem;
      transform: none;
    }
  
    .currency-values {
      gap: 0.5rem;
    }
  
    .currency-values span {
      min-width: 65px;
      font-size: 1.1rem;
    }
  
    .results {
      padding: 0.5rem;
      margin-bottom: 0.5rem;
    }
  
    .result-item {
      font-size: .95rem;
    }
  
    /* Make currency selector more compact */
    .currency-selector {
      gap: 0.5rem;
      font-size: 0.85rem;
      margin: 0 auto;
      text-align: center;
    }
  
    .currency-selector select {
      min-width: auto;
      max-width: 100px;
      margin: 0 auto;
    }
  
    .last-updated {
      font-size: 0.7rem;
      text-align: center;
    }
  
    .input-field input {
      width: 60%;
      margin: 0 auto;
    }
  
    h1 {
      font-size: 1.5rem;
      margin-bottom: 1rem;
    }
  
    .input-group {
      gap: 0.75rem;
    }
  }
  
  /* Extra small screens */
  @media (max-width: 360px) {
    .calculator-container {
      width: 98vw;
      padding: 0.4rem;
    }
  
    .currency-values span {
      min-width: 60px;
    }
  }
  
  .points-move-input {
    display: none;
  }
  
  .points-move:focus + .points-move-input {
    display: block;
    position: absolute;
    width: 100%;
    z-index: 10;
  }
  
  .input-field select {
    padding: 0.25rem;
    border: 1px solid #ddd;
    border-radius: 6px;
    text-align: center;
    font-size: 0.95rem;
    color: black;
    background-color: rgb(244, 244, 244);
    width: 50%;
    height: 32px;
    margin: 0 auto;
  }
  
  /* Specific styling for contracts and points move */
  .points-move,
  select[v-model="contracts"] {
    height: 28px;
    padding: 0 0.25rem;
    min-height: unset;
  }
  
  .input-field input {
    padding: 0.25rem;
    border: 1px solid #ddd;
    border-radius: 6px;
    text-align: center;
    font-size: 0.95rem;
    color: black;
    width: 50%;
    height: 28px;
    margin: 0 auto;
    display: block;
  }
  
  /* Base styles for mobile first */
  .input-field select,
  .input-field input {
    height: 28px;
    padding: 0 0.25rem;
    min-height: unset;
  }
  
  /* Landscape/Desktop adjustments */
  @media (min-width: 768px) {
    .input-field select,
    .input-field input {
      height: 50px;
      padding: 0 0.5rem;
    }

    .points-move,
    select[v-model="contracts"] {
      height: 36px;
    }

    .currency-selector select {
      height: 40px;
    }
    .last-updated {
      font-size: 0.8rem;
      margin-bottom: 1.5rem;
    }
  }
  
  .input-row {
    display: flex;
    gap: 1rem;
    justify-content: center;
    width: 70%;
    margin: 0 auto;
  }
  
  .half-width {
    width: 50%;
  }
  
  .half-width input,
  .half-width select {
    width: 100% !important;
    height: 28px;
    font-size: 0.95rem;
  }
  
  /* Landscape/Desktop adjustments */
  @media (min-width: 768px) {
    .half-width input,
    .half-width select {
      height: 36px;
    }
  }
  
  /* Mobile adjustments */
  @media (max-width: 480px) {
    .input-row {
      gap: 0.7rem;
    }
    
    .half-width input,
    .half-width select {
      font-size: 0.9rem;
    }
  }
  </style>
  
  