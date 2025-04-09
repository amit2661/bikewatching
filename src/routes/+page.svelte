<script>
import mapboxgl from "mapbox-gl";
import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
import * as d3 from "d3";

mapboxgl.accessToken = "pk.eyJ1IjoiYW1pdDI2NjEiLCJhIjoiY205NDdzNXJiMHc1eTJqbjdpZGh4bXl6eSJ9.PT56YvsxsorD0lRJgjHhbA";

import { onMount } from "svelte";

let map;
let stations;
let trips =[];
let departures = new Map();
let arrivals = new Map();


async function initMap() {
	 map = new mapboxgl.Map({
		container: 'map',
		center: [-71.09415, 42.36027],
		zoom: 12,
		style: "mapbox://styles/mapbox/streets-v12",
	});
	await new Promise(resolve => map.on("load", resolve));
	map.addSource("boston_route", {
		type: "geojson",
		data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
	});

    map.addLayer({
	id: "bikelane", // A name for our layer (up to you)
	type: "line", // one of the supported layer types, e.g. line, circle, etc.
	source: "boston_route", // The id we specified in `addSource()`
	paint: {
		"line-color": "green",
		"line-width": 3,
		"line-opacity": 0.4
        // paint params, e.g. colors, thickness, etc.
	},
});

map.addSource("cambridge_route", {
		type: "geojson",
		data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
	});

    map.addLayer({
	id: "Cambridgebikelane", // A name for our layer (up to you)
	type: "line", // one of the supported layer types, e.g. line, circle, etc.
	source: "cambridge_route", // The id we specified in `addSource()`
	paint: {
		"line-color": "green",
		"line-width": 3,
		"line-opacity": 0.4
        // paint params, e.g. colors, thickness, etc.
	},
});
}


onMount(() => {
	initMap();
});

 stations = [];

onMount(async () => {
	stations = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-stations.csv");
    trips = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv").then(trips => {
	for (let trip of trips) {
		trip.started_at = new Date(trip.started_at); // ✅ convert to Date object
			trip.ended_at = new Date(trip.ended_at);
	}
	return trips;
});
    
	departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
	arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);// TODO arrivals

	// Enrich station objects
	stations = stations.map(station => {
		const id = station.Number;
		station.arrivals = arrivals.get(id) ?? 0;
		station.departures = departures.get(id) ?? 0;
		station.totalTraffic = station.arrivals + station.departures;
		return station;
});

});


function getCoords (station) {
	let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
	let {x, y} = map.project(point);
	return {cx: x, cy: y};
}

let mapViewChanged = 0;
$: map?.on("move", evt => mapViewChanged++);

$: radiusScale = d3.scaleSqrt()
	.domain([0, d3.max(filteredStations, d => d.totalTraffic) || 0])
	.range(timeFilter === -1 ? [0, 25] : [3, 30]);


let timeFilter = -1;


$: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
    .toLocaleString("en", {timeStyle: "short"});

	function minutesSinceMidnight (date) {
	return date.getHours() * 60 + date.getMinutes();
}

$: filteredTrips = timeFilter === -1? trips : trips.filter(trip => {
	let startedMinutes = minutesSinceMidnight(trip.started_at);
	let endedMinutes = minutesSinceMidnight(trip.ended_at);
	return Math.abs(startedMinutes - timeFilter) <= 60
	       || Math.abs(endedMinutes - timeFilter) <= 60;
});

// ...
$: filteredDepartures = d3.rollup(filteredTrips, v => v.length, d => d.start_station_id);
$: filteredArrivals = d3.rollup(filteredTrips, v => v.length, d => d.end_station_id);
// ...
$: filteredStations = stations.map(original => {
	let station = { ...original }; // clone the station object
	let id = station.Number;
	station.arrivals = filteredArrivals.get(id) ?? 0;
	station.departures = filteredDepartures.get(id) ?? 0;
	station.totalTraffic = station.arrivals + station.departures;
	return station;
});

let stationFlow = d3.scaleQuantize()
	.domain([0, 1])
	.range([0, 0.5, 1]);


const urlBase = 'https://api.mapbox.com/isochrone/v1/mapbox/';
const profile = 'cycling';
const minutes = [5, 10, 15, 20];
const contourColors = [
	"03045e",
	"0077b6",
	"00b4d8",
	"90e0ef"
]
let isochrone = null;

async function getIso(lon, lat) {
	const base = `${urlBase}${profile}/${lon},${lat}`;
	const params = new URLSearchParams({
		contours_minutes: minutes.join(','),
		contours_colors: contourColors.join(','),
		polygons: 'true',
		access_token: mapboxgl.accessToken
	});
	const url = `${base}?${params.toString()}`;

	const query = await fetch(url, { method: 'GET' });
	isochrone = await query.json();
}

getIso(-71.09415, 42.36027);

let selectedStation = null;

function geoJSONPolygonToPath(feature) {
	const path = d3.path();
	const rings = feature.geometry.coordinates;

	for (const ring of rings) {
		for (let i = 0; i < ring.length; i++) {
			const [lng, lat] = ring[i];
			const { x, y } = map.project([lng, lat]);
			if (i === 0) path.moveTo(x, y);
			else path.lineTo(x, y);
		}
		path.closePath();
	}
	return path.toString();
}


$: if (selectedStation) {
	getIso(+selectedStation.Long, +selectedStation.Lat);
} else {
	isochrone = null;
}


</script>

<header>
    <h1>🚲 BikeWatch</h1>
    <label>
        Filter by time:
        <input type="range" min="-1" max="1440" bind:value={timeFilter} />
        {#if timeFilter !== -1}
            <time style="display: block">
                {timeFilterLabel}
            </time>
        {:else}
            <em style="display: block">(any time)</em>
        {/if}
    </label>
</header>


<div id="map">
    <svg>
        {#key mapViewChanged}
            {#if isochrone}
                {#each isochrone.features as feature}
                    <path
                            d={geoJSONPolygonToPath(feature)}
                            fill={feature.properties.fillColor}
                            fill-opacity="0.2"
                            stroke="#000000"
                            stroke-opacity="0.5"
                            stroke-width="1"
                    >
                        <title>{feature.properties.contour} minutes of biking</title>
                    </path>
                {/each}
            {/if}
            {#each filteredStations as station}
					<circle
						{...getCoords(station)}
						r={radiusScale(station.totalTraffic)}
						fill="steelblue"
						fill-opacity= "60%"
						stroke= "white"
						style="--departure-ratio: { stationFlow(station.departures / station.totalTraffic) }"
						class:selected={station?.Number === selectedStation?.Number}
on:mousedown={() => {
	selectedStation = selectedStation?.Number === station?.Number ? null : station;
}}

					/>
				{/each}
			{/key}
    </svg>
</div>

			
		

<div class="legend">
	<strong>Legend:</strong>
	<div class="legend-item">
		<div class="legend-circle" style="--departure-ratio: 1.0"></div>
		<span>More Departures</span>
	</div>
	<div class="legend-item">
		<div class="legend-circle" style="--departure-ratio: 0.5"></div>
		<span>Balanced</span>
	</div>
	<div class="legend-item">
		<div class="legend-circle" style="--departure-ratio: 0.0"></div>
		<span>More Arrivals</span>
	</div>
</div>


<style>
    @import url("$lib/global.css");

    #map {
	flex: 1;
    background-color: aqua;
    }

	#map svg {
	position: absolute;
	z-index: 1;
	width: 100%;
	height: 100%;
	pointer-events: auto;

}

/* Style for all circles */
#map svg circle {
	transition: opacity 0.2s ease;
	pointer-events: auto;
}

/* Dims other circles when one is selected */
#map svg:has(circle.selected) circle:not(.selected) {
	opacity: 0.3;
}


/* Add pointer events to paths (isochrones) */
#map svg path {
	pointer-events: auto;
}

	
	header {
	display: flex;
	gap: 1em;
	align-items: baseline;
}

label {
	margin-left: auto;
	display: flex;
	flex-direction: column;
}

label time,
label em {
	display: block;
}

label em {
	color: #666;
	font-style: italic;
}

/* Shared color styling */
#map circle,
.legend-circle {
	--color-departures: steelblue;
	--color-arrivals: darkorange;
	--color: color-mix(
		in oklch,
		var(--color-departures) calc(100% * var(--departure-ratio)),
		var(--color-arrivals)
	);
	fill: var(--color);
	background-color: var(--color);
}

/* Legend container */
.legend {
	display: flex;
	flex-wrap: wrap;
	justify-content: center;
	align-items: center;
	gap: 1em;
	margin-block: 1em;
	font-size: 0.95em;
}

/* Individual legend item */
.legend-item {
	display: flex;
	align-items: center;
	gap: 0.5em;
	padding: 0.25em 0.5em;
}

/* Colored circle */
.legend-circle {
	width: 1.2em;
	height: 1.2em;
	border-radius: 50%;
}


</style>