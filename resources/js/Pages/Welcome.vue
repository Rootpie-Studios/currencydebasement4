<script setup>
import { ref, onMounted, onUnmounted } from "vue";
import { Head } from "@inertiajs/vue3";
import * as d3 from "d3";
import * as topojson from "topojson-client";

const hoveredCountry = ref("");
const selectedYear = ref(2017);
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

const euroCountries = [
    "Germany",
    "France",
    "Italy",
    "Spain",
    "Portugal",
    "Ireland",
    "Netherlands",
    "Belgium",
    "Luxembourg",
    "Austria",
    "Finland",
    "Greece",
    "Slovenia",
    "Cyprus",
    "Malta",
    "Slovakia",
    "Estonia",
    "Latvia",
    "Lithuania",
];

const dkkCountries = ["Denmark", "Greenland", "Faroe Islands"];

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
    if (euroCountries.includes(country)) {
        return calculateGainForEuro(country);
    }

    if (dkkCountries.includes(country)) {
        country = "Denmark"; // Use Denmark's data for all DKK countries
    }

    const startPrice = goldData.value[selectedYear.value];
    const currentPrice = goldData.value[2017];

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

function calculateGainForEuro(country) {
    const startPrice = goldData.value[selectedYear.value];
    const currentPrice = goldData.value[2017];

    const euroRates = exchangeRates.value["Euro"];
    if (!euroRates) return null;

    if (selectedYear.value < 1999) return null;

    const startExRate = euroRates[selectedYear.value];
    const endExRate = euroRates["2017"];

    if (!startExRate || !endExRate) return null;

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

function calculateUSDComparison(country) {
    if (usdCountries.includes(country)) return null; // No comparison needed for USD countries

    const startYear = selectedYear.value;
    const endYear = "2017";

    if (euroCountries.includes(country)) {
        const euroRates = exchangeRates.value["Euro"];
        if (!euroRates || !euroRates[startYear] || !euroRates[endYear])
            return null;

        const startRate = euroRates[startYear];
        const endRate = euroRates[endYear];

        const percentageChange = (
            ((1 / endRate - 1 / startRate) / (1 / startRate)) *
            100
        ).toFixed(2);
        return {
            gain: percentageChange,
            loss: Math.abs(percentageChange),
        };
    }

    if (dkkCountries.includes(country)) {
        country = "Denmark";
    }

    const countryRates = exchangeRates.value[country];
    if (!countryRates || !countryRates[startYear] || !countryRates[endYear])
        return null;

    const startRate = countryRates[startYear];
    const endRate = countryRates[endYear];

    const percentageChange = (
        ((1 / endRate - 1 / startRate) / (1 / startRate)) *
        100
    ).toFixed(2);
    return {
        gain: percentageChange,
        loss: Math.abs(percentageChange),
    };
}

function getColorForCountry(country) {
    if (dkkCountries.includes(country)) {
        const countryRates = exchangeRates.value["Denmark"]; // Use Denmark's rates for all DKK countries
        if (!countryRates) return "#d3d3d3";

        const gain = calculateGain("Denmark"); // Use Denmark's calculations
        if (!gain) return "#d3d3d3";

        return gain.gain > 0
            ? `rgb(255, ${255 - Math.min(Math.abs(gain.gain) / 2, 255)}, ${
                  255 - Math.min(Math.abs(gain.gain) / 2, 255)
              })`
            : `rgb(${255 - Math.min(Math.abs(gain.gain) / 2, 255)}, 255, ${
                  255 - Math.min(Math.abs(gain.gain) / 2, 255)
              })`;
    }

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

    // Check if it's a Eurozone country
    if (euroCountries.includes(country)) {
        const euroRates = exchangeRates.value["Euro"];
        if (!euroRates) return "#d3d3d3";

        // Use Euro data for Eurozone countries
        const gain = calculateGainForEuro(country);
        if (!gain) return "#d3d3d3";

        return gain.gain > 0
            ? `rgb(255, ${255 - Math.min(Math.abs(gain.gain) / 2, 255)}, ${
                  255 - Math.min(Math.abs(gain.gain) / 2, 255)
              })`
            : `rgb(${255 - Math.min(Math.abs(gain.gain) / 2, 255)}, 255, ${
                  255 - Math.min(Math.abs(gain.gain) / 2, 255)
              })`;
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
    // Clear existing SVG content
    d3.select("#map").selectAll("*").remove();

    const width = window.innerWidth;
    const height = window.innerHeight - 60;
    const svg = d3
        .select("#map")
        .attr("width", width)
        .attr("height", height)
        .attr("viewBox", [0, 0, width, height]);

    const projection = d3
        .geoNaturalEarth1()
        .scale(Math.min(width / 5.5, height / 3.5))
        .translate([width / 2, height / 2.2]);

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
            .attr("d", d3.geoPath().projection(projection))
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

    // Add resize event listener
    window.addEventListener("resize", () => {
        drawMap();
    });
});

// Clean up resize listener when component is unmounted
onUnmounted(() => {
    window.removeEventListener("resize", drawMap);
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

        <div class="flex items-center gap-2 justify-center py-4">
            <input
                type="range"
                v-model="selectedYear"
                :min="1971"
                :max="2017"
                class="flex-1 max-w-md"
                @input="updateMapColors"
            />
            <span class="text-black dark:text-white w-16 text-right">{{
                selectedYear
            }}</span>
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
                                }}'s currency from {{ selectedYear }} to 2017
                                <br />
                                {{
                                    hoveredCountry ===
                                    "United States of America"
                                        ? "USD"
                                        : hoveredCountry
                                }}'s currency has lost
                                {{ gainPercentage.loss }}% against gold from
                                {{ selectedYear }} to 2017
                            </template>
                            <template v-else>
                                Gold has lost
                                {{ Math.abs(gainPercentage.gain) }}% against
                                {{
                                    hoveredCountry ===
                                    "United States of America"
                                        ? "USD"
                                        : hoveredCountry
                                }}'s currency from {{ selectedYear }} to 2017
                                <br />
                                {{
                                    hoveredCountry ===
                                    "United States of America"
                                        ? "USD"
                                        : hoveredCountry
                                }}'s currency has gained
                                {{ Math.abs(gainPercentage.loss) }}% against
                                gold from {{ selectedYear }} to 2017
                            </template>

                            <template
                                v-if="calculateUSDComparison(hoveredCountry)"
                            >
                                <div
                                    class="mt-2 pt-2 border-t border-gray-200 dark:border-zinc-700"
                                >
                                    <template
                                        v-if="
                                            calculateUSDComparison(
                                                hoveredCountry
                                            ).gain > 0
                                        "
                                    >
                                        {{ hoveredCountry }}'s currency has
                                        gained
                                        {{
                                            calculateUSDComparison(
                                                hoveredCountry
                                            ).gain
                                        }}% against USD from
                                        {{ selectedYear }} to 2017
                                    </template>
                                    <template v-else>
                                        {{ hoveredCountry }}'s currency has lost
                                        {{
                                            calculateUSDComparison(
                                                hoveredCountry
                                            ).loss
                                        }}% against USD from
                                        {{ selectedYear }} to 2017
                                    </template>
                                </div>
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
.map-container {
    width: 100%;
    height: calc(100vh - 60px);
    position: relative;
}
#map {
    width: 100%;
    height: 100%;
    display: block;
}
.tooltip {
    pointer-events: none;
    z-index: 10;
}
blockquote {
    text-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
}
input[type="range"] {
    -webkit-appearance: none;
    height: 8px;
    background: #d3d3d3;
    border-radius: 4px;
    outline: none;
}

input[type="range"]::-webkit-slider-thumb {
    -webkit-appearance: none;
    appearance: none;
    width: 20px;
    height: 20px;
    background: #666;
    border-radius: 50%;
    cursor: pointer;
}

input[type="range"]::-moz-range-thumb {
    width: 20px;
    height: 20px;
    background: #666;
    border-radius: 50%;
    cursor: pointer;
}

.dark input[type="range"] {
    background: #444;
}

.dark input[type="range"]::-webkit-slider-thumb {
    background: #999;
}

.dark input[type="range"]::-moz-range-thumb {
    background: #999;
}
</style>
