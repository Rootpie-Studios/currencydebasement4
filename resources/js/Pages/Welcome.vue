<script setup>
import { ref, onMounted } from "vue";
import { Head } from "@inertiajs/vue3";
import * as d3 from "d3";
import * as topojson from "topojson-client";

const hoveredCountry = ref("");
const selectedYear = ref(1971);
const goldData = ref({});
const exchangeRates = ref({});
const gainPercentage = ref(null);

const usdCountries = [
    "United States of America",
    "Ecuador",
    "El Salvador",
    "Panama",
    "East Timor",
    "British Virgin Islands",
    "Caribbean Netherlands",
    "Turks and Caicos Islands",
    "Zimbabwe", // Note: Multiple currencies used, USD is one of them
    "Marshall Islands",
    "Micronesia",
    "Palau",
];

// Load both datasets
async function loadData() {
    const [goldPrices, rates] = await Promise.all([
        d3.csv("/data/annual_gold_price_usd.csv"),
        d3.csv("/data/annual_exchange_rates.csv"),
    ]);

    // Process gold price data
    goldData.value = goldPrices.reduce((acc, row) => {
        acc[row.Date] = parseFloat(row.Price);
        return acc;
    }, {});

    // Process exchange rates data - fix the column names and date format
    exchangeRates.value = rates.reduce((acc, row) => {
        if (!acc[row.Country]) {
            acc[row.Country] = {};
        }
        // Extract year from the date string
        const year = row.Date.split("-")[0];
        // Use the "Exchange rate" column and parse it
        acc[row.Country][year] = parseFloat(row["Exchange rate"]);
        return acc;
    }, {});
}

function calculateGain(country) {
    const startPrice = goldData.value[selectedYear.value];
    const currentPrice = goldData.value[2024];

    if (usdCountries.includes(country)) {
        const percentageGain = (
            ((currentPrice - startPrice) / startPrice) *
            100
        ).toFixed(2);
        const percentageLoss = (
            ((startPrice - currentPrice) / currentPrice) *
            100
        ).toFixed(2);
        return {
            gain: percentageGain,
            loss: Math.abs(percentageLoss),
        };
    } else {
        const countryRates = exchangeRates.value[country];

        if (!countryRates) {
            return null;
        }

        // Use just the year numbers now
        const startExRate = countryRates[selectedYear.value];
        const endExRate = countryRates["2017"]; // Using 2017 as latest available

        if (!startExRate || !endExRate) {
            return null;
        }

        const startPriceLocal = startPrice * startExRate;
        const endPriceLocal = currentPrice * endExRate;

        const percentageGain = (
            ((endPriceLocal - startPriceLocal) / startPriceLocal) *
            100
        ).toFixed(2);
        const percentageLoss = (
            ((startPriceLocal - endPriceLocal) / endPriceLocal) *
            100
        ).toFixed(2);

        return {
            gain: percentageGain,
            loss: Math.abs(percentageLoss),
        };
    }
}

function getColorForCountry(country) {
    if (usdCountries.includes(country)) {
        const gain = calculateGain(country);
        if (!gain) return "#d3d3d3"; // gray for no data
        return gain.gain > 0
            ? `rgb(255, ${255 - Math.min(Math.abs(gain.gain), 255)}, ${
                  255 - Math.min(Math.abs(gain.gain), 255)
              })` // red for loss
            : `rgb(${255 - Math.min(Math.abs(gain.gain), 255)}, 255, ${
                  255 - Math.min(Math.abs(gain.gain), 255)
              })`; // green for gain
    }

    const countryRates = exchangeRates.value[country];
    if (!countryRates) return "#d3d3d3"; // gray for no data

    const gain = calculateGain(country);
    if (!gain) return "#d3d3d3";

    // Color gradient: red for loss against gold, green for gain
    return gain.gain > 0
        ? `rgb(255, ${255 - Math.min(Math.abs(gain.gain) / 2, 255)}, ${
              255 - Math.min(Math.abs(gain.gain) / 2, 255)
          })` // red
        : `rgb(${255 - Math.min(Math.abs(gain.gain) / 2, 255)}, 255, ${
              255 - Math.min(Math.abs(gain.gain) / 2, 255)
          })`; // green
}

function drawMap() {
    const width = window.innerWidth;
    const height = window.innerHeight - 60;
    const svg = d3.select("#map").attr("width", width).attr("height", height);
    const projection = d3
        .geoNaturalEarth1()
        .scale(Math.min(width / 5.5, height / 3.5))
        .translate([width / 2, height / 2.2]);
    const path = d3.geoPath().projection(projection);

    d3.json(
        "https://cdn.jsdelivr.net/npm/world-atlas@2/countries-110m.json"
    ).then((world) => {
        const countries = topojson.feature(world, world.objects.countries);
        const filteredCountries = {
            type: countries.type,
            features: countries.features.filter(
                (d) => d.properties.name !== "Antarctica"
            ),
        };

        svg.selectAll("path")
            .data(filteredCountries.features)
            .enter()
            .append("path")
            .attr("d", path)
            .attr("fill", (d) => getColorForCountry(d.properties.name))
            .attr("stroke", "#fff")
            .attr("cursor", "pointer")
            .attr("class", "country")
            .on("mouseover", (event, d) => {
                hoveredCountry.value = d.properties.name;
                gainPercentage.value = calculateGain(d.properties.name);
                d3.select(event.currentTarget).attr("fill", (d) => {
                    const baseColor = getColorForCountry(d.properties.name);
                    // Brighten the color slightly on hover
                    return baseColor === "#d3d3d3" ? "#e8e8e8" : baseColor;
                });
            })
            .on("mouseout", (event, d) => {
                hoveredCountry.value = "";
                gainPercentage.value = null;
                d3.select(event.currentTarget).attr("fill", (d) =>
                    getColorForCountry(d.properties.name)
                );
            });
    });
}

function updateMapColors() {
    d3.select("#map")
        .selectAll("path")
        .attr("fill", (d) => getColorForCountry(d.properties.name));
}

onMounted(async () => {
    await loadData();
    drawMap();
});
</script>

<template>
    <Head title="Currency Debasement" />
    <div class="world-map min-h-screen bg-gray-50 dark:bg-black p-4">
        <h1
            class="text-xl font-bold mb-4 text-center text-black dark:text-white"
        >
            Currency Debasement
        </h1>
        <div class="fixed top-4 left-4 flex items-center gap-2">
            <input
                type="range"
                v-model="selectedYear"
                :min="1971"
                :max="2024"
                class="w-48"
                @input="updateMapColors"
            />
            <span class="text-black dark:text-white">{{ selectedYear }}</span>
        </div>
        <div class="map-container relative">
            <svg id="map"></svg>
            <div
                v-if="hoveredCountry"
                class="tooltip fixed top-4 right-4 bg-white dark:bg-zinc-800 border border-gray-200 dark:border-zinc-700 p-3 rounded-lg shadow-lg"
            >
                <h3 class="text-black dark:text-white">
                    {{ hoveredCountry }}
                    <template v-if="gainPercentage">
                        <br />
                        <span class="text-sm">
                            <template v-if="gainPercentage.gain > 0">
                                Gold has gained {{ gainPercentage.gain }}%
                                against
                                {{
                                    hoveredCountry ===
                                    "United States of America"
                                        ? "USD"
                                        : hoveredCountry
                                }}'s currency since {{ selectedYear }}
                                <br />
                                {{
                                    hoveredCountry ===
                                    "United States of America"
                                        ? "USD"
                                        : hoveredCountry
                                }}'s currency has lost
                                {{ gainPercentage.loss }}% against gold since
                                {{ selectedYear }}
                            </template>
                            <template v-else>
                                Gold has lost
                                {{ Math.abs(gainPercentage.gain) }}% against
                                {{
                                    hoveredCountry ===
                                    "United States of America"
                                        ? "USD"
                                        : hoveredCountry
                                }}'s currency since {{ selectedYear }}
                                <br />
                                {{
                                    hoveredCountry ===
                                    "United States of America"
                                        ? "USD"
                                        : hoveredCountry
                                }}'s currency has gained
                                {{ Math.abs(gainPercentage.loss) }}% against
                                gold since {{ selectedYear }}
                            </template>
                        </span>
                    </template>
                </h3>
            </div>
        </div>

        <div
            class="fixed bottom-6 left-1/2 transform -translate-x-1/2 max-w-2xl text-center"
        >
            <blockquote class="text-black/70 dark:text-white/70 italic">
                "The root problem with conventional currency is all the trust
                that's required to make it work. The central bank must be
                trusted not to debase the currency, but the history of fiat
                currencies is full of breaches of that trust."
                <footer
                    class="mt-2 text-black/50 dark:text-white/50 not-italic"
                >
                    — Satoshi Nakamoto
                </footer>
            </blockquote>
        </div>

        <div
            class="fixed bottom-20 left-4 flex flex-col gap-2 bg-white dark:bg-zinc-800 p-2 rounded-lg shadow-lg"
        >
            <div class="text-sm text-black dark:text-white">Legend:</div>
            <div class="flex items-center gap-2">
                <div class="w-4 h-4 bg-red-500"></div>
                <span class="text-xs text-black dark:text-white"
                    >Loss vs Gold</span
                >
            </div>
            <div class="flex items-center gap-2">
                <div class="w-4 h-4 bg-green-500"></div>
                <span class="text-xs text-black dark:text-white"
                    >Gain vs Gold</span
                >
            </div>
            <div class="flex items-center gap-2">
                <div class="w-4 h-4 bg-gray-300"></div>
                <span class="text-xs text-black dark:text-white">No Data</span>
            </div>
        </div>
    </div>
</template>

<style scoped>
.world-map {
    font-family: Arial, sans-serif;
    padding: 0;
    height: 100vh;
    overflow: hidden;
}
#map {
    width: 100%;
    height: calc(100vh - 60px);
    display: block;
}
.tooltip {
    pointer-events: none;
    z-index: 10;
}
blockquote {
    text-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
}
</style>
