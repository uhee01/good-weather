<template>
    <b-modal :show="show" @hide="hideModal" :title="weatherData.city">
        <!-- 로딩 중일 때 모달  -->
        <div v-if="loading" class="text-center p-3">
            <!-- 로딩 스피너 -->
            <div class="spinner-border text-primary mt-3" role="status">
                <span class="visually-hidden"></span>
            </div>
            <p>Loading...</p>
        </div>

        <div v-else class="w-100 max-w-md p-3 bg-white rounded shadow-lg text-center">
            <!-- 로딩 중이 아닐 때 모달  -->
            <div class="flex flex-col items-center">
                <div class="d-flex justify-content-between align-items-start">
                    <h1 class="h2 font-bold text-gray-800">{{ weatherData.city }}</h1>
                    <!-- 닫기 버튼 -->
                    <button type="button" class="btn-close" aria-label="닫기" @click="hideModal"
                        style="align-self: flex-start;"></button>
                </div>
                <!-- 현재 온도, 날씨 -->
                <p class="display-5">{{ weatherData.temperature }}</p>
                <div class="mt-3">
                    <img class="text-gray-800" :src="'/' + weatherData.description + '.png'"
                        style="aspect-ratio: 80 / 80; object-fit: cover; width: 80px; height: 80px;" />
                    <p class="h6 mt-2">{{ weatherData.description }}</p>
                </div>
            </div>

            <!-- 모달 내용 -->
            <div class="modal-content-container"
                style="max-height: 280px; overflow-y: auto; overflow-x: hidden; border-top: 1px solid #dee2e6;">
                <!-- 예보 영역 -->
                <div class="mt-3">
                    <h2 class="h5 font-bold">예보</h2>
                    <div class="row row-cols-1 row-cols-md-3 g-3 mt-2">
                        <!-- 예보 카드 -->
                        <div class="col" v-for="(day, index) in forecast" :key="index">
                            <div class="card bg-light">
                                <div class="card-body">
                                    <p class="small font-medium">{{ day.name }}</p>
                                    <img :src="'/' + day.icon + '.png'" alt="날씨 아이콘" class="text-gray-800"
                                        style="aspect-ratio: 40 / 40; object-fit: cover; width: 40px; height: 40px;" />
                                    <p class="small font-medium">{{ day.temperature }}</p>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>

                <!-- 날씨 상세 영역 -->
                <div class="container mt-5">
                    <div class="row gy-5">
                        <!-- 날씨 상세 카드 -->
                        <div class="col-md-6" v-for="(detail, index) in weatherDetails" :key="index">
                            <div class="card h-100">
                                <div class="card-title-outside">
                                    <h6 class="card-title">{{ detail.title }}</h6>
                                </div>
                                <div class="card-body">
                                    <p class="card-text" v-for="(data, key) in detail.data" :key="key">
                                        <small class="text-muted">{{ data.label }}</small><br>{{ data.value }}
                                    </p>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </b-modal>
</template>

<script lang="ts">
import { Options, Vue } from 'vue-class-component';
import { Prop } from 'vue-property-decorator';
import axios from 'axios';

// 날씨 데이터 인터페이스
interface WeatherData {
    city: string;
    icon: string;
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

// 예보 인터페이스
interface Forecast {
    name: string;
    icon: string;
    temperature: string;
}

// 날씨 상세 데이터 인터페이스
interface WeatherDetail {
    title: string;
    data: { label: string; value: string }[];
}

@Options({})
export default class WeatherModal extends Vue {
    @Prop({ required: true, type: Object }) readonly weatherData!: WeatherData; // weatherData prop을 받아옴
    show = false; // 모달 표시 여부
    loading = false; // 로딩 상태
    forecast: Forecast[] = []; // 예보 데이터
    weatherDetails: WeatherDetail[] = []; // 날씨 상세 데이터

    async mounted() {
        // 컴포넌트가 마운트될 때 실행 (예보데이터, 상세데이터 가져옴)
        await this.fetchForecastData();
        this.prepareWeatherDetails();
    }

    async fetchForecastData() {
        this.loading = true; // 데이터를 가져오는 동안 로딩 상태
        const apiKey = process.env.VUE_APP_WEATHER_API_KEY; // API 키
        const city = this.weatherData.city; // 도시 이름을 weatherData prop에서 가져오기
        const lang = "kr"; 
        const url = `https://api.openweathermap.org/data/2.5/forecast?q=${city}&appid=${apiKey}&lang=${lang}&units=metric`; // API URL

        try {
            const response = await axios.get(url); // API 호출
            const data = response.data; // 응답 데이터

            // '요일' 형식의 날짜로 변환
            this.forecast = data.list.reduce((acc: Forecast[], item: { dt: number; weather: { main: any; }[]; main: { temp_min: number; temp_max: number; }; }) => {
                const date = new Date(item.dt * 1000).toLocaleDateString('ko-KR', { weekday: 'long' }); 

                // 현재 날짜의 예보가 기존에 추가된 예보 데이터에 있는지 확인
                const existingForecast = acc.find(forecast => forecast.name === date);

                if (!existingForecast) {
                    // 기존 예보 데이터가 없으면 새로운 예보 데이터를 추가
                    acc.push({
                        name: date,
                        icon: item.weather[0].main, // 날씨 아이콘
                        temperature: `${item.main.temp_min.toFixed(0)}°C / ${item.main.temp_max.toFixed(0)}°C` // 온도 (최저/최고)
                    });
                } else {
                    // 기존 예보 데이터가 있으면 최저 및 최고 온도를 업데이트
                    const existingMinTemp = parseFloat(existingForecast.temperature.split('°C')[0]); // 기존 최저 온도
                    const existingMaxTemp = parseFloat(existingForecast.temperature.split('°C / ')[1]); // 기존 최고 온도
                    const newMinTemp = item.main.temp_min; // 새로운 최저 온도
                    const newMaxTemp = item.main.temp_max; // 새로운 최고 온도

                    // 최저 온도 업데이트
                    if (!isNaN(newMinTemp) && newMinTemp < existingMinTemp) {
                        existingForecast.temperature = `${newMinTemp.toFixed(0)}°C / ${existingMaxTemp}°C`;
                    }
                    // 최고 온도 업데이트
                    if (!isNaN(newMaxTemp) && newMaxTemp > existingMaxTemp) {
                        existingForecast.temperature = `${existingMinTemp}°C / ${newMaxTemp.toFixed(0)}°C`;
                    }
                }
                return acc; // 예보 데이터 반환
            }, []).slice(1, 4); // 다음 3일간의 예보 데이터 가져오기

        } catch (error) {
            console.error('날씨 예보를 가져오지 못했습니다.', error); 
        }
        this.loading = false; // 로딩 상태 해제
    }

    prepareWeatherDetails() {
        // 날씨 상세 데이터
        this.weatherDetails = [
            {
                title: '일출 & 일몰',
                data: [
                    { label: '일출 시간', value: this.weatherData.sunrise },
                    { label: '일몰 시간', value: this.weatherData.sunset }
                ]
            },
            {
                title: '온도',
                data: [
                    { label: '최저 온도', value: this.weatherData.temp_min },
                    { label: '최고 온도', value: this.weatherData.temp_max },
                    { label: '체감 온도', value: this.weatherData.feels_like }
                ]
            },
            {
                title: '기상 상태',
                data: [
                    { label: '습도', value: this.weatherData.humidity },
                    { label: '기압', value: this.weatherData.pressure }
                ]
            },
            {
                title: '풍향 & 풍속',
                data: [
                    { label: '풍향', value: this.weatherData.deg },
                    { label: '풍속', value: this.weatherData.speed }
                ]
            },
            {
                title: '운량 & 강수량',
                data: [
                    { label: '운량', value: this.weatherData.clouds },
                    { label: '강수량', value: this.weatherData.rain }
                ]
            }
        ];
    }

    hideModal() {
        this.$emit('hideModal'); // 커스텀 이벤트 발생
    }
}
</script>

<style>
.card-title-outside {
    position: relative;
    top: -20px;
    background-color: rgb(199, 238, 255);
    width: calc(100% - 2rem);
    margin-left: auto;
    margin-right: auto;
    box-shadow: 0 2px 4px rgba(0, 0, 0, .2);
    border-radius: .25rem;
    padding-top: 8px;
}
</style>
