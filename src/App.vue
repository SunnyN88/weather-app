<template>
  <div id="app">
    <div class="flowers-wrapper">
      <!-- Sunflower images -->
      <img src="@/assets/sunflower.png" alt="Sunflower" class="left-sunflower" />
      <img src="@/assets/sunflower.png" alt="Sunflower" class="right-sunflower" />

      <div class="container">
        <h1 class="title">Welcome to Sunny's</h1>
        <h1 class="title">Weather App</h1>

        <!-- Searchable Input field -->
        <input
          v-model="city"
          type="text"
          placeholder="Please search for a city..."
          @input="debouncedSearch"
          class="input"
        />

        <!-- Dropdown of city suggestions -->
        <ul v-if="filteredCities.length" class="suggestions-list">
          <li 
            v-for="(suggestion, index) in filteredCities" 
            :key="index" 
            @click="selectCity(suggestion)"
            class="suggestion-item">
            {{ suggestion.name }}
          </li>
        </ul>

        <!-- Display Weather Info -->
        <div v-if="weatherData" class="weather-info">
          <h2>{{ weatherData.city }}</h2>
          <p><strong>Temperature:</strong> {{ weatherData.temperature }}°C</p>
          <p><strong>Condition:</strong> {{ weatherData.weather_description }}</p>
          <p><strong>Air Quality PM10:</strong> {{ airQualityData.pm10 }} µg/m³</p>
          <p><strong>UV Index:</strong> {{ uvIndexData }} (UV exposure level)</p>
          <p><strong>Fine Particle Pollution PM2.5:</strong> {{ airQualityData.pm2_5 }} µg/m³</p>
        </div>

        <!-- Display error message -->
        <div v-if="error" class="error">{{ error }}</div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';
import debounce from 'lodash/debounce';
import { weatherDescriptions } from '@/weatherCodes.js'; // Import weather descriptions
import './App.css';

export default {
  data() {
    return {
      city: '', 
      filteredCities: [],
      searchCache: {},
      weatherData: null,
      airQualityData: null, // Store air quality data
      error: ''
    };
  },
  created() {
    this.debouncedSearch = debounce(this.fetchCitySuggestions, 500);
  },
  methods: {
    async fetchCitySuggestions() {
      if (this.city.trim() === '') {
        this.filteredCities = [];
        return;
      }

      if (this.searchCache[this.city]) {
        this.filteredCities = this.searchCache[this.city];
        return;
      }

      try {
        const response = await axios.get(
          `https://nominatim.openstreetmap.org/search?format=json&q=${this.city}&featuretype=city&addressdetails=1&limit=5`
        );

        this.filteredCities = response.data.map(entry => ({
          name: entry.display_name,
          lat: entry.lat,
          lon: entry.lon
        }));

        this.searchCache[this.city] = this.filteredCities;
      } catch (error) {
        console.error("Error fetching city suggestions:", error);
        this.filteredCities = [];
      }
    },

    async selectCity(city) {
      this.city = city.name;
      this.filteredCities = [];

      await this.getWeather(city.lat, city.lon);
    },

    async getWeather(lat, lon) {
      try {
        // Fetch weather data
        const weatherResponse = await axios.get(
          `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&current_weather=true&precipitation_sum&hourly=uv_index`
        );

        // Fetch air quality data
        const airQualityResponse = await axios.get(
          `https://air-quality-api.open-meteo.com/v1/air-quality?latitude=${lat}&longitude=${lon}&hourly=pm10,pm2_5`
        );

        // Extract weather data
        const weatherCode = weatherResponse.data.current_weather.weathercode;

        // Update weather data
        this.weatherData = {
          city: this.city,
          temperature: weatherResponse.data.current_weather.temperature,
          weather_description: weatherDescriptions[weatherCode] || "Unknown condition"
        };

        // Update air quality data
        this.airQualityData = {
          pm10: airQualityResponse.data.hourly.pm10 ? airQualityResponse.data.hourly.pm10[0] : 'N/A',
          pm2_5: airQualityResponse.data.hourly.pm2_5 ? airQualityResponse.data.hourly.pm2_5[0] : 'N/A'
        };

        // Update UV index data
        this.uvIndexData = weatherResponse.data.hourly.uv_index ? weatherResponse.data.hourly.uv_index[0] : 'N/A';

        this.error = '';
      } catch (error) {
        console.error('Error fetching data:', error);
        this.error = 'Error fetching data';
        this.weatherData = null;
        this.airQualityData = null;
      }
    }
  }
};
</script>

<style>
/* Styles go in css file*/
</style>
