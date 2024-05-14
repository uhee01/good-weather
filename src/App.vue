<template>
  <div
    class="background-dark text-white min-vh-100 d-flex flex-column align-items-center justify-content-center p-4 position-relative">
    <!-- 검색창 영역 -->
    <div class="position-relative w-100 max-w-md">
      <input class="form-control rounded-pill ps-4 pe-5 py-2 text-dark bg-light" placeholder="Enter the city name" type="search"
        v-model="searchQuery" @keyup.enter="searchWeather" />
      <span class="bi bi-search position-absolute top-50 end-0 translate-middle-y me-3 text-secondary"
        style="cursor: pointer;" @click="searchWeather"></span>
    </div>
    <!-- 로딩 중일 때 로딩 스피너 표시 -->
    <div v-if="loading" class="spinner-border text-light mt-3" role="status">
      <span class="visually-hidden">Loading...</span>
    </div>
    <!-- 로딩 중이 아닐 때 날씨 정보 카드 표시 (날씨 정보 카드를 표시하는 영역) -->
    <div v-else class="d-grid gap-3 mt-3" style="grid-template-columns: repeat(2, 1fr);">
      <!-- 날씨 데이터 카드 -->
      <b-card class="background-secondary card-hover" v-for="(data, index) in weatherData" :key="index"
        @click="showModal(data)">
        <b-card-body class="d-flex flex-column align-items-center justify-content-center gap-2 p-4">
          <!-- 도시명 -->
          <p class="fs-5 fw-medium">{{ data.city }}</p>
          <!-- 날씨 아이콘 -->
          <img :src="'/' + data.description + '.png'" alt="Weather Icon" style="width: 60px; height: 60px;">
          <!-- 온도 -->
          <p class="fs-3 fw-bold">{{ data.temperature }}</p>
        </b-card-body>
      </b-card>
    </div>
    <!-- 날씨 모달 창 -->
    <WeatherModal :weatherData="selectedData" v-if="showWeatherModal" :show="showWeatherModal"
      @hideModal="showWeatherModal = false" class="position-absolute top-50 start-50 translate-middle modal-bg" />
  </div>
</template>

<script lang="ts">
import { Options, Vue } from 'vue-class-component';
import WeatherModal from './components/WeatherModal.vue';
import axios from 'axios';

// 날씨 데이터 인터페이스
interface WeatherData {
  city: string;
  description: string;
  temperature: string;
  feels_like: string;
  temp_min: string;
  temp_max: string;
  humidity: string;
  pressure: string;
  deg: string;
  speed: string;
  sunrise: string;
  sunset: string;
  clouds: string;
  rain: string;
}

@Options({
  components: {
    WeatherModal
  },
})
export default class App extends Vue {
  // 날씨 데이터
  weatherData: WeatherData[] = [];
  // 선택된 날씨 데이터
  selectedData: WeatherData | null = null;
  // 날씨 모달 창 표시 여부
  showWeatherModal: boolean = false;
  // 로딩 상태
  loading: boolean = false;
  // 검색어
  searchQuery: string = '';



  // 풍향을 문자열로 변환하는 메서드
  convertDegToDirection(deg: number): string {
    if ((deg >= 0 && deg <= 22.5) || (deg > 337.5 && deg <= 360)) {
      return '북풍';
    } else if (deg > 22.5 && deg <= 67.5) {
      return '북동풍';
    } else if (deg > 67.5 && deg <= 112.5) {
      return '동풍';
    } else if (deg > 112.5 && deg <= 157.5) {
      return '남동풍';
    } else if (deg > 157.5 && deg <= 202.5) {
      return '남풍';
    } else if (deg > 202.5 && deg <= 247.5) {
      return '남서풍';
    } else if (deg > 247.5 && deg <= 292.5) {
      return '서풍';
    } else if (deg > 292.5 && deg <= 337.5) {
      return '북서풍';
    } else {
      return '알 수 없음';
    }
  }

  // 날씨 데이터 가져오는 메서드
  async fetchWeatherData() {
    this.loading = true; // 데이터 로딩 중 상태 설정
    this.weatherData = []; // 날씨 데이터 배열 초기화
    const cities = ['Seoul, KR', 'Yongin, KR', 'Seongnam, KR', 'Suwon, KR'];
    const apiKey = 'ba6e7f437ef24e66d3b8cd15a7278f9e';
    for (const city of cities) {
      const url = `https://api.openweathermap.org/data/2.5/weather?q=${city}&appid=${apiKey}&units=metric`;
      try {
        const response = await axios.get(url);
        const data = response.data;
        this.weatherData.push({
          city: data.name,
          description: data.weather[0].main,
          temperature: `${data.main.temp.toFixed(1)}°C`,
          feels_like: `${data.main.feels_like.toFixed(1)}°C`,
          temp_min: `${data.main.temp_min.toFixed(1)}°C`,
          temp_max: `${data.main.temp_max.toFixed(1)}°C`,
          humidity: `${data.main.humidity}%`,
          pressure: `${data.main.pressure}hpa`,
          deg: this.convertDegToDirection(data.wind.deg),
          speed: `${data.wind.speed}m/s`,
          sunrise: new Date(data.sys.sunrise * 1000).toLocaleTimeString(),
          sunset: new Date(data.sys.sunset * 1000).toLocaleTimeString(),
          clouds: `${data.clouds.all}%`,
          rain: data.rain ? `${data.rain['1h']}mm` : '0mm',
        });
      } catch (error) {
        console.error('날씨 정보를 불러오는 데 실패했습니다.', error);
      }
    }
    this.loading = false; // 데이터 로딩 완료 상태 설정
  }

  // 검색 이벤트 핸들러
  async searchWeather() {
    this.loading = true; // 데이터 로딩 중 상태 설정

    // 검색어가 비어 있을 경우 기본 도시 목록에 대한 날씨 데이터를 불러옴
    if (this.searchQuery.trim() === '') {
      await this.fetchWeatherData(); // 기존 도시 목록에 대한 날씨 데이터 가져오기
      this.loading = false; // 데이터 로딩 완료 상태 설정
      return; // 처리 종료
    }

    // 검색어가 한글일 경우에만 영어로 도시 이름을 입력받는 알림창 표시
    if (/^[가-힣]+$/.test(this.searchQuery.trim())) {
      const confirmResult = window.confirm("영어로 입력해주세요.");
      if (!confirmResult) {
        this.loading = false; // 데이터 로딩 완료 상태 설정
        return; // 처리 종료
      }
    }

    const apiKey = 'ba6e7f437ef24e66d3b8cd15a7278f9e';
    const url = `https://api.openweathermap.org/data/2.5/weather?q=${this.searchQuery}&appid=${apiKey}&units=metric`;

    try {
      const response = await axios.get(url);
      const data = response.data;
      this.weatherData = [{
        city: data.name,
        description: data.weather[0].main,
        temperature: `${data.main.temp.toFixed(1)}°C`,
        feels_like: `${data.main.feels_like.toFixed(1)}°C`,
        temp_min: `${data.main.temp_min.toFixed(1)}°C`,
        temp_max: `${data.main.temp_max.toFixed(1)}°C`,
        humidity: `${data.main.humidity}%`,
        pressure: `${data.main.pressure}hpa`,
        deg: this.convertDegToDirection(data.wind.deg),
        speed: `${data.wind.speed}m/s`,
        sunrise: new Date(data.sys.sunrise * 1000).toLocaleTimeString(),
        sunset: new Date(data.sys.sunset * 1000).toLocaleTimeString(),
        clouds: `${data.clouds.all}%`,
        rain: data.rain ? `${data.rain['1h']}mm` : '0mm',
      }];
    } catch (error) {
      console.error('날씨 정보를 불러오는 데 실패했습니다.', error);
    }

    this.loading = false; // 데이터 로딩 완료 상태 설정
  }


  // 날씨 모달 창 표시 메서드
  showModal(data: WeatherData) {
    this.selectedData = data;
    this.showWeatherModal = true;
  }

  // 컴포넌트가 마운트되었을 때 실행되는 메서드
  async mounted() {
    await this.fetchWeatherData();
  }
}
</script>

<style>
.background-dark {
  background-color: #100C2A;
}

.background-secondary {
  background-color: rgb(57, 58, 79);
}

.max-w-md {
  max-width: 28rem;
}

.modal-bg {
  background: white;
  color: black;
}

.card-hover {
  border-radius: 10px;
  transition: transform 0.3s ease, filter 0.3s ease;
}

.card-hover:hover {
  transform: scale(1.05);
  filter: brightness(1.4);
}
</style>