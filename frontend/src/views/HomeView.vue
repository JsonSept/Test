<template>
  <div>
    <h1>Solar Irradiance Calculator</h1>
    <form @submit.prevent="handleSubmit">
      <input
        type="text"
        v-model="address"
        placeholder="Enter your address or suburb"
        required
      />
      <button type="submit">Get Solar Irradiance</button>
    </form>

    <div v-if="loading">
      <p>Loading...</p>
    </div>

    <div v-else-if="error">
      <p>Error: {{ error }}</p>
    </div>

    <div v-else-if="irradiance !== null">
      <p>Location: {{ locationName }}</p>
      <p>Solar Irradiance: {{ irradiance.toFixed(2) }} W/m²</p>
    </div>
  </div>
</template>

<script>
import axios from 'axios';

const API_KEY = 'e945d7f71eb0e5e621a7dfcce2cb1a43';

export default {
  data() {
    return {
      address: '',
      irradiance: null,
      locationName: '',
      loading: false,
      error: null,
    };
  },
  methods: {
    async handleSubmit() {
      if (!this.address) {
        this.error = 'Please enter an address or suburb.';
        return;
      }

      this.loading = true;
      this.error = null;
      this.irradiance = null;

      try {
        // Step 1: Geocode the address to get coordinates
        const geocodeData = await this.getCoordinates(this.address);
        if (geocodeData.length === 0) {
          throw new Error('Location not found.');
        }

        const { lat, lon, name, country, state } = geocodeData[0];
        this.locationName = `${name}${state ? ', ' + state : ''}, ${country}`;

        // Step 2: Fetch weather data using the coordinates
        const weatherData = await this.getWeatherData(lat, lon);

        // Step 3: Calculate solar irradiance
        this.irradiance = this.calculateSolarIrradiance(weatherData);
      } catch (err) {
        console.error(err);
        this.error = err.message || 'An error occurred.';
      } finally {
        this.loading = false;
      }
    },
    async getCoordinates(address) {
      const geocodeUrl = `https://api.openweathermap.org/geo/1.0/direct?q=${encodeURIComponent(
        address
      )}&limit=1&appid=${API_KEY}`;
      const response = await axios.get(geocodeUrl);
      return response.data;
    },
    async getWeatherData(lat, lon) {
      const weatherUrl = `https://api.openweathermap.org/data/2.5/weather?lat=${lat}&lon=${lon}&appid=${API_KEY}`;
      const response = await axios.get(weatherUrl);
      return response.data;
    },
    calculateSolarIrradiance(weatherData) {
      try {
        // Extract necessary data
        const cloudCover = weatherData.clouds.all / 100; // Convert percentage to fraction
        const solarConstant = 1361; // Solar constant in W/m²

        // Calculate solar zenith angle based on current time and location
        // Note: Accurate zenith angle calculation requires more complex logic.
        // Here, we'll use a simplified approach.

        const currentTime = weatherData.dt; // Current time in Unix UTC
        const sunrise = weatherData.sys.sunrise; // Sunrise time in Unix UTC
        const sunset = weatherData.sys.sunset; // Sunset time in Unix UTC

        // Calculate the solar elevation angle (simplified)
        // This is a very rough approximation and may not be accurate.
        let solarElevationAngle;
        if (currentTime < sunrise || currentTime > sunset) {
          // Sun is below the horizon
          solarElevationAngle = -1; // Night time
        } else {
          // Daytime: calculate an approximate solar elevation angle
          const dayProgress = (currentTime - sunrise) / (sunset - sunrise); // 0 at sunrise, 1 at sunset
          // Assuming the sun reaches the highest point at midday (dayProgress = 0.5)
          solarElevationAngle = 90 * Math.sin(dayProgress * Math.PI);
        }

        if (solarElevationAngle <= 0) {
          // Sun is below the horizon; irradiance is zero
          return 0;
        }

        // Clear sky irradiance
        const clearSkyIrradiance = solarConstant * Math.sin(Math.PI * solarElevationAngle / 180);

        // Adjust for cloud cover
        const irradiance = clearSkyIrradiance * (1 - cloudCover);

        return irradiance;
      } catch (error) {
        console.error('Error calculating solar irradiance:', error);
        return null;
      }
    },
  },
};
</script>

<style scoped>
/* Add some basic styling */
form {
  margin-bottom: 20px;
}

input {
  padding: 8px;
  width: 300px;
  margin-right: 10px;
}

button {
  padding: 8px 16px;
}
</style>
