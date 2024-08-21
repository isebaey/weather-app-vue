<template>
  <div class="container my-5">
    <div class="weather-wrapper">
      <div
        class="card p-5 rounded-4 glassmorphic-bg text-center weather-app border-0"
      >
        <h2 class="card-title mb-4">Weather App</h2>
        <p class="card-text mb-4">
          Type a city name and press Enter to search for the weather.
        </p>

        <div class="input-group mb-5 justify-content-center">
          <input
            v-model="query"
            type="text"
            class="form-control form-control-lg holographic-input"
            placeholder="Search for a city"
            @keydown.enter="fetchWeather"
            :aria-describedby="queryError ? 'queryError' : null"
          />
          <div class="input-group-append">
            <button
              class="btn btn-primary btn-lg holographic-btn"
              @click="fetchWeather"
            >
              Search
            </button>
          </div>
        </div>

        <div
          v-if="queryError"
          class="alert alert-warning text-center"
          role="alert"
          id="queryError"
        >
          {{ queryError }}
        </div>

        <div
          v-if="loading"
          class="text-center my-4 d-flex justify-content-center align-items-center"
        >
          <div class="cinematic-loader" role="status"></div>
          <span class="sr-only">Loading...</span>
        </div>

        <div v-if="error" class="alert alert-danger text-center" role="alert">
          {{ error }}
        </div>

        <div
          v-if="weather && !loading && !error"
          class="text-center weather-display"
        >
          <h3 class="display-6 mb-3">
            {{ weather.city.name }}, {{ weather.city.country }}
          </h3>
          <p>{{ dateBuilder() }}</p>
          <img
            :src="weatherIcon(weather?.list[0]?.weather[0]?.icon)"
            class="my-3 weather-icon"
            alt="Weather Icon"
          />
          <h1 class="display-3 mb-3">
            {{ Math.round(weather.list[0].main.temp) }}°{{ temperatureUnit }}
          </h1>
          <p class="lead mb-4">{{ weather.list[0].weather[0].main }}</p>

          <!-- Unit Toggle -->
          <div class="btn-group mb-4" role="group">
            <button
              class="btn btn-secondary"
              @click="toggleUnit('metric')"
              :class="{ active: temperatureUnit === 'C' }"
            >
              °C
            </button>
            <button
              class="btn btn-secondary"
              @click="toggleUnit('imperial')"
              :class="{ active: temperatureUnit === 'F' }"
            >
              °F
            </button>
          </div>

          <!-- 3-Day Forecast -->
          <div class="forecast mt-5">
            <h4 class="mb-4">3-Day Forecast</h4>
            <div class="row justify-content-center">
              <div
                class="col-md-4"
                v-for="(day, index) in forecastDays"
                :key="index"
              >
                <div
                  class="card p-3 mb-4 forecast-card text-center flex align-items-center"
                >
                  <p>{{ forecastDate(day.dt) }}</p>
                  <img
                    :src="weatherIcon(day?.weather[0]?.icon)"
                    class="my-3 forecast-icon"
                    alt="Weather Icon"
                  />
                  <h5>{{ Math.round(day.main.temp) }}°{{ temperatureUnit }}</h5>
                  <p>{{ day.weather[0].main }}</p>
                </div>
              </div>
            </div>
          </div>

          <!-- Average Weekly Temperature Chart -->
          <div class="chart-container mt-5 p-2">
            <h4 class="mb-4">Average Weekly Temperature</h4>
            <canvas id="weeklyTemperatureChart"></canvas>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from "axios";
import { nextTick } from "vue";
import Chart from "chart.js/auto";

export default {
  data() {
    return {
      api_key: "241c8ed1ea03cab07795f86e37bee936",
      url_base: "https://api.openweathermap.org/data/2.5/",
      query: "",
      weather: null,
      forecast: null,
      loading: false,
      error: null,
      chart: null,
      queryError: null,
      temperatureUnit: "C", // Default to Celsius
    };
  },
  computed: {
    forecastDays() {
      if (this.weather) {
        return this.weather.list
          .filter((item, index) => index % 8 === 0)
          .slice(1, 4);
      }
      return [];
    },
  },
  methods: {
    async fetchWeather() {
      if (!this.query) {
        this.queryError = "Please enter a city name.";
        return;
      }

      this.queryError = null;
      this.loading = true;
      this.error = null;
      this.weather = null;

      try {
        const response = await axios.get(
          `${this.url_base}forecast?q=${this.query}&units=${
            this.temperatureUnit === "C" ? "metric" : "imperial"
          }&APPID=${this.api_key}`
        );
        this.weather = response.data;
      } catch (err) {
        this.error = "City not found. Please try again.";
      } finally {
        this.loading = false;
      }
    },
    dateBuilder() {
      let d = new Date();
      let months = [
        "January",
        "February",
        "March",
        "April",
        "May",
        "June",
        "July",
        "August",
        "September",
        "October",
        "November",
        "December",
      ];
      let days = [
        "Sunday",
        "Monday",
        "Tuesday",
        "Wednesday",
        "Thursday",
        "Friday",
        "Saturday",
      ];

      let day = days[d.getDay()];
      let date = d.getDate();
      let month = d.getMonth();
      let year = d.getFullYear();

      return `${day}, ${date} ${months[month]} ${year}`;
    },
    forecastDate(dt) {
      const d = new Date(dt * 1000);
      const months = [
        "January",
        "February",
        "March",
        "April",
        "May",
        "June",
        "July",
        "August",
        "September",
        "October",
        "November",
        "December",
      ];

      const date = d.getDate();
      const month = months[d.getMonth()];

      return `${date} ${month}`;
    },
    weatherIcon(iconCode) {
      return `http://openweathermap.org/img/wn/${iconCode}@2x.png`;
    },
    renderChart() {
      nextTick(() => {
        const ctx = document
          .getElementById("weeklyTemperatureChart")
          .getContext("2d");
        const data = this.getWeeklyTemperatureData();

        if (this.chart) {
          this.chart.destroy();
        }

        this.chart = new Chart(ctx, {
          type: "line",
          data: {
            labels: data.labels,
            datasets: [
              {
                label: "Temperature (°C)",
                data: data.temps,
                borderColor: "#007bff",
                backgroundColor: "rgba(0, 123, 255, 0.1)",
                fill: true,
                tension: 0.4,
              },
              {
                label: "Average Temperature",
                data: data.avgTemps,
                borderColor: "#ff5733",
                backgroundColor: "rgba(255, 87, 51, 0.1)",
                borderDash: [10, 5],
                fill: false,
              },
            ],
          },
          options: {
            responsive: true,
            plugins: {
              legend: {
                display: true,
              },
            },
            scales: {
              y: {
                beginAtZero: false,
              },
            },
          },
        });
      });
    },
    getWeeklyTemperatureData() {
      if (!this.weather) return { labels: [], temps: [], avgTemps: [] };

      const dailyData = this.weather.list.filter(
        (item, index) => index % 8 === 0
      );

      // Collect temperature data
      const temps = dailyData.map((item) => Math.round(item.main.temp));

      // Calculate average temperature
      const avgTemp = temps.reduce((sum, temp) => sum + temp, 0) / temps.length;

      // Generate chart data
      return {
        labels: dailyData.map((item) => this.forecastDate(item.dt)),
        temps: temps,
        avgTemps: new Array(temps.length).fill(avgTemp),
      };
    },
    toggleUnit(unit) {
      this.temperatureUnit = unit === "metric" ? "C" : "F";
      if (this.weather) {
        this.fetchWeather();
      }
    },
  },
  watch: {
    weather(newWeather) {
      if (newWeather) {
        this.renderChart();
      }
    },
  },
  mounted() {
    this.renderChart();
  },
};
</script>

<style scoped>
/* General Styling */
body {
  background: linear-gradient(to bottom, #e9ecef, #f8f9fa);
  font-family: "Poppins", sans-serif;
}

/* Glassmorphism Card */
.glassmorphic-bg {
  background: rgba(255, 255, 255, 0.75);
  backdrop-filter: blur(10px);
  border-radius: 1.5rem;
  box-shadow: 0px 6px 24px rgba(0, 0, 0, 0.1);
}

/* Media Query for Larger Screens */
@media (min-width: 998px) {
  .weather-app {
    min-width: 998px;
  }
}

/* Holographic Effects */
.holographic-input,
.holographic-btn,
.btn.active {
  background: linear-gradient(135deg, #ffcc00, #cc00ff) !important;
  color: #fff !important;
  text-shadow: 0 0 5px rgba(0, 0, 0, 0.3) !important;
  border: none !important;
}

.holographic-btn:hover {
  border: none;
  background: linear-gradient(135deg, #ff9933, #cc33ff);
}

/* Cinematic Loading Animation */
.cinematic-loader {
  width: 4rem;
  height: 4rem;
  border: 0.5rem solid rgba(255, 255, 255, 0.3); /* Light border */
  border-top: 0.5rem solid #ffcc00; /* Holographic yellow */
  border-radius: 50%;
  animation: cinematicSpin 1s linear infinite;
}

@keyframes cinematicSpin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}

/* Weather Icon */
.weather-icon {
  animation: iconFloat 3s ease-in-out infinite;
  max-width: 80px;
}

.forecast-icon {
  max-width: 50px;
}

/* Forecast Styling */
.forecast-card {
  border: none !important;
  padding: 15px;
  color: #343a40;
}

/* Floating Icon Animation */
@keyframes iconFloat {
  0%,
  100% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(-10px);
  }
}

/* Chart Container */
.chart-container {
  border: none !important;
  padding: 20px;
  color: #343a40;
  width: 100%;
}

/* Additional Styling */
.weather-wrapper {
  display: flex;
  justify-content: center;
}

.input-group {
  max-width: 400px;
  margin: auto;
}

.forecast {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.forecast .row {
  justify-content: space-around;
}

.forecast .col-md-4 {
  margin-bottom: 20px;
}

/* Active Button Style */
.btn-group {
  border: none;
  background-color: white;
  color: #fff;
}
</style>
