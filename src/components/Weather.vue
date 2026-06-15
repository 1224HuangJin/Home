<template>
  <div class="weather-wrapper" v-if="weatherData.city && weatherData.data.type">
    <div class="static-side">
      <span class="city-name">{{ displayCity }}</span>
      <span class="divider">|</span>
    </div>

    <div class="carousel-container">
      <transition name="slide-fade">
        <div v-if="step === 0" :key="0" class="weather-item">
          <span class="index-tag">温度</span>
          <span class="weather-icon">{{ extractEmoji(weatherData.data.type) }}</span>
          <span class="temp" :style="{ color: getTempColor(weatherData.data.temp) }">
            {{ weatherData.data.temp }}°C
          </span>
        </div>

        <div v-else-if="step === 1" :key="1" class="weather-item">
          <span class="index-tag">天气状况</span>
          <span class="weather-type">{{ weatherData.data.type }}</span>
        </div>

        <div v-else :key="2" class="weather-item">
          <span class="index-tag">风力级别</span>
          <span class="wind-full">
            {{ weatherData.data.fengxiang }} | {{ weatherData.data.windSpeed }} km/h
          </span>
        </div>
      </transition>
    </div>
  </div>
  <div class="weather-error" v-else>
    <span>加载天气中...</span>
  </div>
</template>

<script setup>
import { reactive, onMounted, onUnmounted, ref, computed } from "vue";
import axios from "axios";

const weatherData = reactive({
  city: null,
  data: { type: null, temp: null, fengxiang: null, windSpeed: null },
});

const displayCity = computed(() => {
  const name = weatherData.city || "";
  return name.length > 10 ? name.slice(0, 10) + "..." : name;
});

const step = ref(0);
let timer = null;

const getTempColor = (temp) => {
  if (temp <= 0) return "#1E90FF";    
  if (temp <= 10) return "#00BFFF";   
  if (temp <= 22) return "#50C878";   
  if (temp <= 30) return "#FFCC00";   
  if (temp <= 38) return "#FF8C00";   
  if (temp <= 50) return "#FF4500";   
  return "#8B0000";                   
};

const extractEmoji = (type) => type?.match(/[\uD800-\uDBFF][\uDC00-\uDFFF]|\u2600|\u2601|\u26C5/)?.[0] || "🌡️";

// 补全了 Open-Meteo (WMO) 标准的所有天气代码
const weatherMap = {
  0: "☀️ 晴朗", 
  1: "🌤️ 多云", 2: "⛅️ 局部多云", 3: "☁️ 阴云密布", 
  45: "🌫 雾", 48: "🌫 霾/沙尘", 
  51: "🌧️ 细雨", 53: "🌧️ 毛毛雨", 55: "🌧️ 强毛毛雨",
  56: "🌧️ 冰冻细雨", 57: "🌧️ 强冰冻细雨",
  61: "🌧 小雨", 63: "🌧 中雨", 65: "🌧 大雨", 
  66: "🌧 冰冻雨", 67: "🌧 强冰冻雨",
  71: "🌨 小雪", 73: "🌨 中雪", 75: "🌨 大雪", 77: "🌨 雪粒",
  80: "🌧️ 阵雨", 81: "🌧️ 强阵雨", 82: "🌧️ 暴雨", 
  85: "🌨 阵雪", 86: "🌨 强阵雪", 
  95: "⛈️ 雷阵雨", 96: "⛈️ 伴冰雹雷阵雨", 99: "⛈️ 强冰雹雷阵雨"
};

const getWindDirLabel = (deg) => {
  const dirs = ["北", "东北", "东", "东南", "南", "西南", "西", "西北"];
  return dirs[Math.round(deg / 45) % 8];
};

// 核心逻辑：通过经纬度获取天气
const fetchWeatherByCoords = async (lat, lon, cityName) => {
  try {
    const res = await axios.get(`https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&current_weather=true`);
    if (res.data?.current_weather) {
      const now = res.data.current_weather;
      weatherData.city = cityName;
      weatherData.data = {
        type: weatherMap[now.weathercode] || "未知",
        temp: Math.round(now.temperature),
        fengxiang: getWindDirLabel(now.winddirection),
        windSpeed: Math.round(now.windspeed), // 添加了圆整，UI更美观
      };
    }
  } catch (error) {
    console.error("天气数据获取失败:", error);
  }
};

// 降级方案：使用更准确的备用 IP 定位
const fetchByIP = async () => {
  try {
    // 换用了 ipwho.is，支持 HTTPS 且部分欧洲节点解析更好，失败则退回 ipapi
    const ipRes = await axios.get("https://ipwho.is/?lang=zh-CN");
    if (ipRes.data.success) {
      await fetchWeatherByCoords(ipRes.data.latitude, ipRes.data.longitude, ipRes.data.city || "未知");
    } else {
      const fallback = await axios.get("https://ipapi.co/json/");
      await fetchWeatherByCoords(fallback.data.latitude, fallback.data.longitude, fallback.data.city || "未知");
    }
  } catch (err) {
    console.error("IP定位完全失败:", err);
  }
};

// 主控函数：优先 GPS，失败走 IP
const getWeatherData = async () => {
  if (navigator.geolocation) {
    navigator.geolocation.getCurrentPosition(
      async (position) => {
        const { latitude, longitude } = position.coords;
        try {
          // 使用完全免费的 BigDataCloud 反向地理编码获取城市名，带中文参数
          const geoRes = await axios.get(`https://api.bigdatacloud.net/data/reverse-geocode-client?latitude=${latitude}&longitude=${longitude}&localityLanguage=zh`);
          const city = geoRes.data.city || geoRes.data.locality || "当前位置";
          await fetchWeatherByCoords(latitude, longitude, city);
        } catch (e) {
          // 哪怕城市名解析失败，也要用经纬度去拉取本地天气！
          await fetchWeatherByCoords(latitude, longitude, "当前位置");
        }
      },
      (error) => {
        console.warn("未获取浏览器定位授权，降级使用 IP 定位");
        fetchByIP();
      },
      { timeout: 5000 } // 避免无限等待
    );
  } else {
    fetchByIP();
  }
};

onMounted(() => {
  getWeatherData();
  timer = setInterval(() => { step.value = (step.value + 1) % 3; }, 4000);
});

onUnmounted(() => clearInterval(timer));
</script>

<style scoped>
/* 你的原有样式完全保持不变，只需确保 .weather-item 设置了 absolute 即可 */
.weather-wrapper {
  display: inline-flex;
  height: 30px;
  align-items: center;
  padding: 0 12px;
  color: #fff;
  font-family: 'Segoe UI', system-ui, sans-serif;
  overflow: hidden; 
}
/* ... 剩下保留你的原有CSS ... */
.static-side { display: flex; align-items: center; flex-shrink: 0; }
.city-name { font-size: 14px; font-weight: 700; color: #ffffff; white-space: nowrap; }
.divider { margin: 0 10px; color: rgba(255, 255, 255, 0.2); user-select: none; }
.carousel-container { position: relative; display: inline-flex; align-items: center; width: 160px; height: 100%; flex-shrink: 0; }
.weather-item { position: absolute; left: 0; display: flex; align-items: center; white-space: nowrap; }
.index-tag { font-size: 9px; background: #007aff; color: #fff; padding: 1px 5px; border-radius: 3px; margin-right: 8px; font-weight: 800; flex-shrink: 0; }
.temp { font-weight: 700; font-size: 15px; }
.weather-type, .wind-full { font-size: 13px; color: #efefef; font-weight: 500; }
.slide-fade-enter-active, .slide-fade-leave-active { transition: all 0.5s ease; }
.slide-fade-enter-from { transform: translateY(12px); opacity: 0; }
.slide-fade-leave-to { transform: translateY(-12px); opacity: 0; }
</style>