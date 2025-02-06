<script setup>
import { ref, onMounted, computed } from "vue";
import { supabase } from "@/supabase";
import * as d3 from "d3";
import * as topojson from "topojson-client";

const startYear = 1972;
const currentYear = new Date().getFullYear();
const selectedYear = ref(startYear);
const currencyData = ref({});
const currencyValues = ref({});
const hoveredCountry = ref("");
const hoveredCurrency = ref("");

const years = computed(() =>
    Array.from({ length: currentYear - startYear + 1 }, (_, i) => startYear + i)
);

async function fetchData() {
    const [currencies, values] = await Promise.all([
        supabase.from("currencies").select("*"),
        supabase
            .from("currency_values")
            .select("currencies(code), year, value_in_usd")
            .gte("year", startYear)
            .lte("year", currentYear),
    ]);

    if (currencies.error)
        console.error("Error fetching currencies:", currencies.error);
    if (values.error)
        console.error("Error fetching currency values:", values.error);

    currencyData.value = currencies.data.reduce((acc, c) => {
        acc[c.country] = { currency: c.name, code: c.code };
        return acc;
    }, {});

    currencyValues.value = values.data.reduce((acc, v) => {
        if (!acc[v.currencies.code]) acc[v.currencies.code] = {};
        acc[v.currencies.code][v.year] = v.value_in_usd;
        return acc;
    }, {});
}

function calculateAppreciation(code) {
    if (code === "USD" || !currencyValues.value[code]) return null;
    const startValue = currencyValues.value[code][selectedYear.value];
    const endValue = currencyValues.value[code][currentYear];
    if (!startValue || !endValue) return null;
    return (((endValue - startValue) / startValue) * 100).toFixed(2);
}

function getCurrencyInfo(countryName) {
    const info = currencyData.value[countryName];
    if (!info) return null;
    const appreciation = calculateAppreciation(info.code);
    return {
        ...info,
        appreciation: appreciation !== null ? parseFloat(appreciation) : null,
    };
}

function drawMap() {
    const width = 960;
    const height = 500;
    const svg = d3.select("#map").attr("width", width).attr("height", height);
    const projection = d3
        .geoNaturalEarth1()
        .scale(153)
        .translate([width / 2, height / 1.5]);
    const path = d3.geoPath().projection(projection);

    d3.json(
        "https://cdn.jsdelivr.net/npm/world-atlas@2/countries-110m.json"
    ).then((world) => {
        const countries = topojson.feature(world, world.objects.countries);

        svg.selectAll("path")
            .data(countries.features)
            .enter()
            .append("path")
            .attr("d", path)
            .attr("fill", (d) => {
                const info = getCurrencyInfo(d.properties.name);
                return info && info.appreciation !== null
                    ? "#4a7a6c"
                    : "#d3d3d3";
            })
            .attr("stroke", "#fff")
            .on("mouseover", (event, d) => {
                const info = getCurrencyInfo(d.properties.name);
                hoveredCountry.value = d.properties.name;
                if (info && info.appreciation !== null) {
                    const appreciationText =
                        info.appreciation > 0
                            ? `Depreciated by ${info.appreciation}%`
                            : `Appreciated by ${Math.abs(info.appreciation)}%`;
                    hoveredCurrency.value = `${info.currency} (${info.code}): ${appreciationText} vs USD since ${selectedYear.value}`;
                } else {
                    hoveredCurrency.value = "Currency data not available";
                }
            })
            .on("mouseout", () => {
                hoveredCountry.value = "";
                hoveredCurrency.value = "";
            });
    });
}

function handleYearChange() {
    drawMap();
}

onMounted(async () => {
    await fetchData();
    drawMap();
});
</script>

<template>
    <div class="currency-debasement">
        <div class="controls">
            <label for="year-slider">Start year: {{ selectedYear }}</label>
            <input
                type="range"
                id="year-slider"
                v-model="selectedYear"
                :min="startYear"
                :max="currentYear"
                :step="1"
            />
            <button @click="handleYearChange">Update</button>
        </div>
        <svg id="map"></svg>
        <div v-if="hoveredCountry" class="tooltip">
            <h3>{{ hoveredCountry }}</h3>
            <p>{{ hoveredCurrency }}</p>
        </div>
    </div>
</template>

<style scoped>
.currency-debasement {
    font-family: Arial, sans-serif;
}
.controls {
    margin-bottom: 20px;
}
#map {
    max-width: 100%;
    height: auto;
}
.tooltip {
    position: absolute;
    background: white;
    border: 1px solid #ddd;
    padding: 10px;
    pointer-events: none;
}
</style>
