<template>
  <div class="weather" v-if="weatherData.adCode.city && weatherData.weather.weather">
    <span>{{ weatherData.adCode.city }}&nbsp;</span>
    <span>{{ weatherData.weather.weather }}&nbsp;</span>
    <span>{{ weatherData.weather.temperature }}℃</span>
    <span class="sm-hidden">
      &nbsp;{{
        weatherData.weather.winddirection?.endsWith("风")
          ? weatherData.weather.winddirection
          : weatherData.weather.winddirection + "风"
      }}&nbsp;
    </span>
    <span class="sm-hidden">{{ weatherData.weather.windpower }}&nbsp;级</span>
  </div>
  <div class="weather" v-else>
    <span>天气数据获取失败</span>
  </div>
</template>

<script setup>
import { getAdcode, getWeather, getOtherWeather } from "@/api";
import { ElMessage } from "element-plus";
import { h } from "vue";
import { Error } from "@icon-park/vue-next";

// 高德开发者 Key
const mainKey = import.meta.env.VITE_WEATHER_KEY || "2dc8fcb6b3fec61561562016ac78c197";

// 太原市默认配置
const DEFAULT_CITY = {
  city: "太原市",
  adcode: "141000" // 太原市行政区划代码
};

// 天气数据
const weatherData = reactive({
  adCode: { ...DEFAULT_CITY },
  weather: {
    weather: null,
    temperature: null,
    winddirection: null,
    windpower: null
  }
});

// 错误处理函数
const handleError = (message) => {
  ElMessage.error({ message, icon: h(Error) });
  console.error("[天气服务错误]", message);
};

// 获取天气数据（增强版）
const getWeatherData = async () => {
  try {
    // 优先使用配置的 Key
    if (!mainKey) {
      console.log("未配置 Key，使用备用接口");
      const result = await getOtherWeather();
      weatherData.adCode = {
        city: result.cityName || DEFAULT_CITY.city,
        adcode: result.adcode || DEFAULT_CITY.adcode
      };
      weatherData.weather = parseWeatherData(result);
      return;
    }

    // 尝试获取城市信息
    let adCode = await getAdcode(mainKey);
    
    // 回退逻辑：若获取失败则使用默认值
    if (!adCode || adCode.infocode !== "10000") {
      console.warn("城市查询失败，自动切换至太原市");
      adCode = { city: DEFAULT_CITY.city, adcode: DEFAULT_CITY.adcode };
    }

    // 获取天气信息（带重试机制）
    let retryCount = 0;
    const maxRetries = 2;
    while (retryCount < maxRetries) {
      const weatherRes = await getWeather(mainKey, adCode.adcode);
      if (weatherRes.status === "1") {
        weatherData.weather = parseWeatherData(weatherRes);
        return;
      }
      retryCount++;
      await new Promise(resolve => setTimeout(resolve, 1000 * retryCount)); // 延迟重试
    }

    throw new Error("天气数据获取失败，请检查网络或稍后重试");

  } catch (error) {
    handleError(error.message);
    // 最终回退：显示默认城市天气
    weatherData.adCode = { ...DEFAULT_CITY };
    weatherData.weather = {
      weather: "晴",    // 默认天气
      temperature: "20",// 默认温度
      winddirection: "无持续风向",
      windpower: "微风"
    };
  }
};

// 数据解析工具函数
const parseWeatherData = (res) => {
  return {
    weather: res.lives?.[0]?.weather || "晴",
    temperature: res.lives?.[0]?.temperature || "20",
    winddirection: res.lives?.[0]?.winddirection || "无持续风向",
    windpower: res.lives?.[0]?.windpower || "微风"
  };
};

onMounted(() => {
  getWeatherData();
});
</script>
