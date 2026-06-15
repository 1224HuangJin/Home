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

// 降级方案：多重 IP 定位保障链（A 计划失败自动走 B 计划，B 失败走 C 计划）
const fetchByIP = async () => {
  // === 【第一重保障：Cloudflare 原生高精度地理信息】 ===
  // 专门解决西班牙/欧洲区域的定位，不需要任何 Key，完全免费且极速
  try {
    console.log("正在尝试第一重 IP 定位 (Cloudflare)...");
    const cfRes = await axios.get("https://1.1.1.1/cdn-cgi/trace");
    // Cloudflare 返回的是文本数据，形如：loc=ES\nip=xxx.xxx.xxx.xxx
    const locMatch = cfRes.data.match(/loc=([A-Z]+)/);
    const ipMatch = cfRes.data.match(/ip=([^\n]+)/);
    
    if (locMatch && ipMatch) {
      const countryCode = locMatch[1];
      const userIP = ipMatch[1];
      
      // 如果定位在西班牙 (ES)，我们可以顺藤摸瓜用另一个稳定接口拿到城市
      const geoRes = await axios.get(`https://ipapi.co/${userIP}/json/`);
      if (geoRes.data && geoRes.data.latitude) {
        console.log("第一重定位成功：", geoRes.data.city);
        await fetchWeatherByCoords(geoRes.data.latitude, geoRes.data.longitude, geoRes.data.city || "巴塞罗那");
        return; // 成功则直接退出，不再走后续接口
      }
    }
  } catch (cfErr) {
    console.warn("第一重 IP 定位失败，正在切换第二重...", cfErr);
  }

  // === 【第二重保障：ipapi.co 官方直接接口】 ===
  try {
    console.log("正在尝试第二重 IP 定位 (ipapi.co)...");
    const fallback = await axios.get("https://ipapi.co/json/");
    if (fallback.data && fallback.data.latitude) {
      await fetchWeatherByCoords(fallback.data.latitude, fallback.data.longitude, fallback.data.city || "未知城市");
      return;
    }
  } catch (ipapiErr) {
    console.warn("第二重 IP 定位也失败了，正在切换最终兜底...", ipapiErr);
  }

  // === 【第三重保障：终极盲盒兜底】 ===
  // 如果所有 IP 接口全部把你封了或者网络全断，直接硬编码默认位置（既然你在巴塞罗那，就拿巴塞罗那当默认值）
  console.error("所有 IP 定位接口全部沦陷！触发巴塞罗那硬编码兜底。");
  const FORCE_LAT = 41.3851;
  const FORCE_LON = 2.1734;
  await fetchWeatherByCoords(FORCE_LAT, FORCE_LON, "Barcelona");
};

// 主控函数：优先依靠高精度 GPS/Wi-Fi，失败则立刻进入 IP 链
const getWeatherData = async () => {
  if (navigator.geolocation) {
    navigator.geolocation.getCurrentPosition(
      async (position) => {
        const { latitude, longitude } = position.coords;
        try {
          // 使用大厂公共逆地理编码
          const geoRes = await axios.get(`https://api.bigdatacloud.net/data/reverse-geocode-client?latitude=${latitude}&longitude=${longitude}&localityLanguage=zh`);
          const city = geoRes.data.city || geoRes.data.locality || "当前位置";
          await fetchWeatherByCoords(latitude, longitude, city);
        } catch (e) {
          // 哪怕城市名解析失败，经纬度是对的，直接强行查天气
          await fetchWeatherByCoords(latitude, longitude, "当前位置");
        }
      },
      (error) => {
        // 这里不要用 console.warn 阻塞，直接无缝切入 IP 定位
        console.log("浏览器未授权或获取超时，无缝切换至 IP 定位链...");
        fetchByIP();
      },
      { 
        enableHighAccuracy: false, // 改为 false 能够极大加快非手机端的 Wi-Fi 定位速度
        timeout: 4000,             // 4秒不响应立刻降级，不让用户久等
        maximumAge: 60000          // 允许使用 1 分钟内的缓存定位，提高加载速度
      }
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
