<script>
import axios from 'axios';
import dayjs from 'dayjs';
axios.defaults.baseURL = "http://localhost:3005"

import { Line  } from 'vue-chartjs'
import { Chart as ChartJS, Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale, LineElement, PointElement } from 'chart.js'

ChartJS.register(
    Title, Tooltip, Legend, 
    CategoryScale, LinearScale, 
    BarElement, LineElement, PointElement
)

export default {
    name: 'BarChart',
    components: { Line  },

    data() {
        return {
            counter: 0,
            graphLabels: [],
            graphData: {},
            min_coef: 5,
            max_coef: 2.6,
            products: [],
            plants: [],
            user: {},
            selectedProducts: [],
            quantity: 0,
            resultCost: 0,
            priceHistory: [],
            timeInterval: 0,
            datasets: [],
            chartOptions: {
                responsive: true
            }
        };
    },

    computed: {
        chartData() {
            return this.datasets.map(dataset => ({
                label: dataset.label,
                data: dataset.data,
                backgroundColor: this.getRandomColor(),
                borderColor: this.getRandomColor(),
            }));
        }
    },


    methods: {
        async loadShop() {
            let response = await axios.get("/shop");
            this.products = response.data;
            this.selectedProducts = new Array(response.data.length).fill(false);
            this.initializeDatasets();
        },

        async loadUser() {
            let responce = await axios.get("/user");
            this.user = responce.data;
        },

        initializeDatasets() {
            this.datasets = this.products.map(product => ({
                label: product.title,
                data: [],
                backgroundColor: this.getRandomColor(),
                borderColor: this.getRandomColor(),
                borderWidth: 2,
                tension: 0.1, // Убедитесь, что у вас это значение не слишком высокое
                fill: false,
            }));
        },


        async loadData() {
            await this.loadShop();
            this.products.forEach(product => {
                this.graphData[product.title] = [];
            });
            await this.loadPriceHistory(); // Загрузка цен
        },

        async loadGarden() {
            let responce = await axios.get("/garden");
            this.plants = responce.data;
        },

        async loadPriceHistory() {
            let response = await axios.get("/price-history");
            this.priceHistory = response.data;
            this.updateGraphDataWithPriceHistory();
        },

        randomCost() {
            if (this.products) {
                for (let i = 0; i < this.products.length; i++) {
                    const newPrice = Math.floor(Math.random() * this.max_coef * this.products[i].buyPrice + this.min_coef);
                    this.products[i].buyPrice = newPrice;
                }
            }
        },
        
        async plant(event) {
            event.preventDefault();
            if (this.user.gold >= this.resultCost) {
                let listRes = [];
                for (let i = 0; i < this.products.length; i++) {
                    if (this.selectedProducts[i]) {
                        const responce = await axios.post("/shop/buy", {
                            title: this.products[i].title,
                            count: this.quantity,
                        });
                        listRes.push(responce);
                    }
                }

                const allSuccess = listRes.every(res => res.data === 'ok');

                if (allSuccess) {
                    console.log("Вы успешно приобрели растения!");
                } else {
                    console.error("Ошибка при покупке растений!");
                }
            } else {
                console.error("Недостаточно средств для покупки!");
            }

            console.log(this.products)
            this.products.forEach((product, index) => {
                this.datasets.data.push(product.buyPrice);
                console.log(this.datasets)
            });

            this.selectedProducts = new Array(this.products.length).fill(false);
            this.quantity = 0;
            this.resultCost = 0;
        },

        updateGraphDataWithPriceHistory() {
            this.priceHistory.forEach(entry => {
                const timestamp = dayjs(entry.timestamp).format('HH:mm:ss');
                this.graphLabels.push(timestamp);
                
                const dataset = this.datasets.find(d => d.label === entry.title);
                if (dataset) {
                    dataset.data.push(entry.price);
                }
            });
        },

        changeQuantity() {
            this.resultCost = 0;
            if (this.quantity > 0) {
                for (let i = 0; i < this.products.length; i++) {
                    if (this.selectedProducts[i]) this.resultCost += this.products[i].buyPrice * this.quantity;
                }
            }
        },

        isHarvest(plant) {
            return dayjs() < dayjs(plant.timeToHarvest);
        },

        getPlantTime(plant) {
            let plantMinut = 0;
            let plantSecond = 0;
            if (this.isHarvest(plant)) {
                plantMinut = (dayjs(plant.timeToHarvest).minute() - dayjs().minute());
                plantSecond = (dayjs(plant.timeToHarvest).second() - dayjs().second());
                return dayjs().set("minute", plantMinut).set("second", plantSecond).format('mm:ss');
            } else return "Готово";
        },

        async harvestPlant(plant) {
            let responce = await axios.post("/garden/harvest", {
                id: plant._id,
            });
            this.loadUser();
            this.loadGarden();
        },
        
        getRandomColor() {
            const letters = '0123456789ABCDEF';
            let color = '#';
            for (let i = 0; i < 6; i++) {
                color += letters[Math.floor(Math.random() * 16)];
            }
            return color;
        }
    },

    mounted() {
        this.loadGarden();
        this.loadUser();
        this.loadShop();
        this.loadData();
        setInterval(() => {
            this.timeInterval++;
            this.loadGarden();
            if (this.counter % 3 == 0) {
                this.randomCost();
            }
            this.counter++;
        }, 1000);
    }
}
</script>

<template>
    <div class="app">
        <div class="container my-3">
            <form class="shop">
                <div class="row top-row">
                    <div class="col">
                        <div class="shop-plants">
                            <label class="shop-item" v-for="(product, index) in products" :key="index">
                                <img :src="'src/assets/' + product.image + '.png'">
                                <p> {{ product.title }} </p>
                                <p> {{ product.buyPrice }} </p>
                                <input type="checkbox" v-model="selectedProducts[index]">
                                <p> {{ index }}</p>
                            </label>
                        </div>
                    </div>
                    <div class="col">
                        <div class="shop-form">
                            <div class="shop-info-row">
                                <div class="shop-info-left">
                                    <label for="count" class="form-label">Количество:</label>
                                </div>
                                <div class="shop-info-right">
                                    <input type="number" id="count" class="form-control count-input" v-model="quantity" @change="changeQuantity()">
                                </div>
                            </div>
                            <div class="shop-info-row">
                                <div class="shop-info-left">
                                    Итого: 
                                </div>
                                <div class="shop-info-right">
                                    $ {{ resultCost }}
                                </div>
                            </div>

                            <div class="shop-info-row">
                                <div class="shop-info-left">
                                    Ваш баланс:
                                </div>
                                <div class="shop-info-right">
                                    $ {{ user.gold }}
                                </div>
                            </div>

                            <div class="shop-info-row">
                                <div class="shop-info-left">
                                </div>
                                <div class="shop-info-right">
                                    <button type="submit" class="btn btn-primary" @click="plant">
                                        Посадить
                                    </button>
                                </div>
                            </div>
                        </div>
                        <div class="combined-chart">
                            <h3>Динамика цен всех продуктов</h3>
                            <Line
                                :key="'combined-chart-' + graphLabels.length"
                                :options="chartOptions"
                                :data="{
                                    labels: graphLabels,
                                    datasets: datasets
                                }"
                            />
                        </div>
                    </div>
                </div>
            </form>

            <div class="garden">
                <div class="garden-plant" v-for="plant in plants" @click="harvestPlant(plant)">
                    <img :src="'src/assets/' + plant.image + '.png'">
                    <p> {{ getPlantTime(plant) }} </p>
                </div>
            </div>
        </div>
    </div>
</template>

<style>
.shop {
    border: 4px solid black;
    padding: 10px;
}

.top-row {
    padding: 20px;
}

.shop-info-row {
    display: flex;
    align-items: center;
    font-size: 20px;
    gap: 20px;
    height: 40px;
    margin: 20px;
}
.shop-info-left {
    width: 120px;
    text-align: right;
}
.shop-info-right {
    width: 100px;
}
.total {
    font-size: 20px;
    height: 40px;
    display: flex;
    align-items: center;
}

.count-input {
    font-size: 20px;
}

.btn-primary {
    font-size: 20px;
}

.user-info {
    font-size: 30px;
    padding-left: 40px;
    padding-right: 10px;
    text-align: right;
}

.shop-plants {
    display: flex;
    flex-wrap: wrap;
    justify-content: flex-start;
}

.shop-item {
    width: 150px;
    padding: 20px;
    text-align: center;
}

.shop-item img {
    width: 100%;
}

.garden {
    border: 4px solid black;
    padding: 20px;
    display: flex;
    flex-wrap: wrap;
    background-image: url('src/assets/background.jpg');
    background-size: cover;
    min-height: 300px;
}

.garden-plant {
    width: 150px;
    padding: 20px;
    text-align: center;
    color: white;
}

.garden-plant img {
    width: 100%;
}
</style>
