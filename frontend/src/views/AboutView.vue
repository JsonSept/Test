<template>
  <div class="solar-calculator bg-dark text-light">
    <h1>Solar Power Calculator</h1>

    <!-- Input for Location -->
    <div class="form-group">
      <label for="location">Your Location</label>
      <input type="text" id="location" v-model="location" placeholder="Enter your city or town">
    </div>

    <!-- Simplified Panel Area Input -->
    <div class="form-group">
      <label for="panelArea">Solar Array Surface Area (m²)</label>
      <input type="number" id="panelArea" v-model="panelArea" placeholder="e.g., 10">
    </div>

    <!-- Simplified Panel Efficiency with Default -->
    <div class="form-group">
      <label for="efficiency">Panel Efficiency</label>
      <select id="efficiency" v-model="efficiency">
        <option value="15">Standard (15%)</option>
        <option value="18">High Efficiency (18%)</option>
        <option value="20">Premium (20%)</option>
      </select>
    </div>

    <!-- Button to Calculate -->
    <button class="btn" @click="calculatePower">Calculate Power Output</button>

    <!-- Display the Results -->
    <div v-if="powerGenerated !== null">
      <h2 class="out text-light">Estimated Power Output: {{ powerGenerated.toFixed(2) }} kWh/day</h2>
      <p class="out text-light">This is equivalent to {{ powerInWatts.toFixed(2) }} W/day</p>
      <p class="out text-light">(This is an estimate based on average sunlight hours for your location)</p>
    </div>

    <!-- Input for Daily Consumption -->
    <div class="form-group" v-if="powerGenerated !== null">
      <label for="consumption">Your Daily Consumption (kWh/day)</label>
      <input type="number" id="consumption" v-model="dailyConsumption" placeholder="e.g., 5">
    </div>

    <!-- Button to Compare with Consumption -->
    <button class="btn" v-if="powerGenerated !== null && dailyConsumption !== null" @click="compareConsumption">Compare Consumption</button>

    <!-- Display the Consumption Comparison and Currency Calculation -->
    <div v-if="consumptionResult !== null">
      <h3 class="out text-light">{{ consumptionResult }}</h3>
      <h4 v-if="savingsInRand !== null" class="out text-light">
        Potential Savings: R{{ savingsInRand.toFixed(2) }}
      </h4>
    </div>
  </div>
</template>

<script>
import axios from 'axios';

const apiKey = 'e945d7f71eb0e5e621a7dfcce2cb1a43';
// const GEOCODING_API_KEY = 'YOUR_OPENCAGE_API_KEY'; // Replace with your OpenCage API key

function getWeatherData(apiKey, lat, lon) {
    const url = `http://api.openweathermap.org/data/2.5/weather?lat=${lat}&lon=${lon}&appid=${apiKey}`;
    return axios.get(url)
        .then(response => response.data)
        .catch(error => { throw new Error('Error fetching weather data'); });
}

function calculateSolarIrradiance(weatherData) {
    const cloudCover = weatherData.clouds.all / 100;
    const solarConstant = 1361;
    const zenithAngle = 90 - weatherData.sys.sunrise;

    const clearSkyIrradiance = solarConstant * Math.cos(zenithAngle * (Math.PI / 180));
    const irradiance = clearSkyIrradiance * (1 - cloudCover);
    
    return irradiance;
}

// function getCoordinates(address) {
//     const url = `https://api.opencagedata.com/geocode/v1/json?q=${encodeURIComponent(address)}&key=${GEOCODING_API_KEY}`;
//     return axios.get(url)
//         .then(response => {
//             if (response.data.results.length > 0) {
//                 const { lat, lng } = response.data.results[0].geometry;
//                 return { lat, lon: lng };
//             } else {
//                 throw new Error('Address not found');
//             }
//         })
//         .catch(error => { throw new Error('Error fetching coordinates'); });
// }

export default {
    data() {
        return {
            location: '',
            panelArea: 10, // Default value
            efficiency: 18, // Default value (in percentage)
            sunlightHours: 5, // Default estimated average sunlight hours
            powerGenerated: null,
            powerInWatts: null,
            dailyConsumption: null,
            consumptionResult: null,
            savingsInRand: null,
            conversionRate: 1.33, // Conversion rate
            randPerKWh: 1.5, // Example Rand rate per kWh
            irradiance: null,
            loading: false,
            error: null,
        };
    },
    methods: {
        async calculatePower() {
            this.loading = true;
            this.error = null;
            try {
                const { lat, lon } = await getCoordinates(this.location);
                const weatherData = await getWeatherData(WEATHER_API_KEY, lat, lon);
                this.irradiance = calculateSolarIrradiance(weatherData);
                const panelEfficiencyDecimal = this.efficiency / 100;

                this.powerGenerated = (this.panelArea * this.irradiance * panelEfficiencyDecimal * this.sunlightHours) / 1000;

                this.powerInWatts = this.powerGenerated * 1000;
            } catch (error) {
                this.error = error.message;
            } finally {
                this.loading = false;
            }
        },
        compareConsumption() {
            if (this.dailyConsumption > this.powerGenerated) {
                this.consumptionResult = `Your daily consumption of ${this.dailyConsumption} kWh exceeds the generated power of ${this.powerGenerated.toFixed(2)} kWh/day.`;
            } else {
                this.consumptionResult = `Your generated power of ${this.powerGenerated.toFixed(2)} kWh/day is sufficient for your daily consumption of ${this.dailyConsumption} kWh.`;

                const differenceInKWh = this.powerGenerated - this.dailyConsumption;
                const savings = differenceInKWh * this.conversionRate;
                this.savingsInRand = savings * this.randPerKWh;
            }
        }
    }
};
</script>

<style>
body {
  background-color: rgb(72, 66, 66);
}
/* Styles remain the same */
.solar-calculator {
  max-width: 700px;
  margin: auto;
  padding: 20px;
  border-radius: 10px;
  box-shadow: 0px 0px 10px rgba(0, 0, 0, 0.1);
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  justify-content: space-between;
  padding-top: 50px;
  padding-bottom: 100px;
}

.form-group {
  margin-bottom: 15px;
  flex: 1 1 calc(50% - 20px);
  min-width: 250px;
}

label {
  font-weight: bold;
  display: block;
}

input,
select,
.btn {
  width: 100%;
  padding: 10px;
  margin-top: 5px;
  border-radius: 5px;
}

.btn {
  background: #ff5722;
  color: white;
  font-size: 16px;
  cursor: pointer;
}

.btn:hover {
  background-color: #ff5725;
}

h2,
h3,
h4 {
  margin: 0;
  padding: 10px 0;
}

.out {
  flex: 1 1 calc(100% - 20px);
}

@media (min-width: 768px) {
  .out {
    flex: 1 1 calc(50% - 20px);
  }
}
</style>
