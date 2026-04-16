<script>
	import { onMount } from "svelte";
	import * as d3 from "d3";
	import mapboxgl from "mapbox-gl";
	import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";

	mapboxgl.accessToken = "pk.eyJ1IjoiY291cnRuZXltYSIsImEiOiJjbW8xcG13emwwbXJ0MnFxYXpmOXhzb3MyIn0.6ZynnBjAMtGSNyRQ8lPqBw";

	let map;
	let stations = [];
	let trips = [];

	let arrivals;
	let departures;

	let filteredArrivals = new Map();
	let filteredDepartures = new Map();
	let filteredStations = [];

	let radiusScale;

	let mapViewChanged = 0;

	let timeFilter = -1;
	$: timeFilterLabel =
		timeFilter === -1
			? ""
			: new Date(0, 0, 0, 0, timeFilter).toLocaleString("en", {
					timeStyle: "short"
				});

	const departuresByMinute = Array.from({ length: 1440 }, () => []);
	const arrivalsByMinute = Array.from({ length: 1440 }, () => []);

	let selectedStation = null;
	let isochrone = null;

	const stationFlow = d3.scaleQuantize().domain([0, 1]).range([0, 0.5, 1]);

	async function loadStations() {
		stations = await d3.csv(
			"https://vis-society.github.io/labs/9/data/bluebikes-stations.csv"
		);
	}

	function minutesSinceMidnight(date) {
		return date.getHours() * 60 + date.getMinutes();
	}

	async function loadTrips() {
		trips = await d3
			.csv("https://vis-society.github.io/labs/9/data/bluebikes-traffic-2024-03.csv")
			.then((trips) => {
				for (let trip of trips) {
					trip.started_at = new Date(trip.started_at);
					trip.ended_at = new Date(trip.ended_at);

					let startedMinutes = minutesSinceMidnight(trip.started_at);
					let endedMinutes = minutesSinceMidnight(trip.ended_at);

					departuresByMinute[startedMinutes].push(trip);
					arrivalsByMinute[endedMinutes].push(trip);
				}
				return trips;
			});

		departures = d3.rollup(trips, (v) => v.length, (d) => d.start_station_id);
		arrivals = d3.rollup(trips, (v) => v.length, (d) => d.end_station_id);

		stations = stations.map((station) => {
			let id = station.Number;
			station.arrivals = arrivals.get(id) ?? 0;
			station.departures = departures.get(id) ?? 0;
			station.totalTraffic = station.arrivals + station.departures;
			return station;
		});
	}

	function filterByMinute(tripsByMinute, minute) {
		let minMinute = (minute - 60 + 1440) % 1440;
		let maxMinute = (minute + 60) % 1440;

		if (minMinute > maxMinute) {
			let beforeMidnight = tripsByMinute.slice(minMinute);
			let afterMidnight = tripsByMinute.slice(0, maxMinute);
			return beforeMidnight.concat(afterMidnight).flat();
		} else {
			return tripsByMinute.slice(minMinute, maxMinute).flat();
		}
	}

	function getCoords(station) {
		let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
		let { x, y } = map.project(point);
		return { cx: x, cy: y };
	}

	async function getIso(lon, lat) {
		const base = `https://api.mapbox.com/isochrone/v1/mapbox/cycling/${lon},${lat}`;
		const params = new URLSearchParams({
			contours_minutes: [5, 10, 15, 20].join(","),
			contours_colors: ["03045e", "0077b6", "00b4d8", "90e0ef"].join(","),
			polygons: "true",
			access_token: mapboxgl.accessToken
		});
		const url = `${base}?${params.toString()}`;

		const query = await fetch(url, { method: "GET" });
		isochrone = await query.json();
	}

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

	async function initializeMap() {
		map = new mapboxgl.Map({
			container: "map",
			style: "mapbox://styles/mapbox/streets-v12",
			center: [-71.09415, 42.36027],
			zoom: 12
		});

		await new Promise((resolve) => map.on("load", resolve));

		map.addSource("boston_route", {
			type: "geojson",
			data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D"
		});

		map.addLayer({
			id: "boston_bikelanes",
			type: "line",
			source: "boston_route",
			paint: {
				"line-color": "rgb(34, 197, 94)",
				"line-width": 3,
				"line-opacity": 0.4
			}
		});

		map.addSource("cambridge_route", {
			type: "geojson",
			data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson"
		});

		map.addLayer({
			id: "cambridge_bikelanes",
			type: "line",
			source: "cambridge_route",
			paint: {
				"line-color": "rgb(34, 197, 94)",
				"line-width": 3,
				"line-opacity": 0.4
			}
		});

		map.on("move", () => mapViewChanged++);
	}

	onMount(async () => {
		await loadStations();
		await loadTrips();
		await initializeMap();
	});

	$: filteredDepartures =
		timeFilter === -1
			? departures ?? new Map()
			: d3.rollup(
					filterByMinute(departuresByMinute, timeFilter),
					(v) => v.length,
					(d) => d.start_station_id
				);

	$: filteredArrivals =
		timeFilter === -1
			? arrivals ?? new Map()
			: d3.rollup(
					filterByMinute(arrivalsByMinute, timeFilter),
					(v) => v.length,
					(d) => d.end_station_id
				);

	$: filteredStations = stations.map((station) => {
		const id = station.Number;
		const arr = filteredArrivals.get(id) ?? 0;
		const dep = filteredDepartures.get(id) ?? 0;
		return {
			...station,
			arrivals: arr,
			departures: dep,
			totalTraffic: arr + dep
		};
	});

	$: radiusScale = d3
		.scaleSqrt()
		.domain([0, d3.max(filteredStations, (d) => d.totalTraffic) || 0])
		.range(timeFilter === -1 ? [0, 25] : [3, 30]);

	$: if (selectedStation) {
		getIso(+selectedStation.Long, +selectedStation.Lat);
	} else {
		isochrone = null;
	}
</script>

<svelte:head>
	<title>BikeWatch</title>
</svelte:head>

<header>
	<h1>🚲 BikeWatch</h1>
	<label>
		Filter by time:
		<input type="range" min="-1" max="1440" bind:value={timeFilter} />
		{#if timeFilter !== -1}
			<time>{timeFilterLabel}</time>
		{:else}
			<em>(any time)</em>
		{/if}
	</label>
</header>

<p>
	An interactive map of BlueBike traffic in the Boston and Cambridge area.
</p>

<div id="map">
	<svg>
		{#if map}
			{#key mapViewChanged}
				{#if isochrone}
					{#each isochrone.features as feature}
						<path
							d={geoJSONPolygonToPath(feature)}
							fill={"#" + feature.properties.fillColor}
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
						style={`--departure-ratio: ${stationFlow(
							station.totalTraffic ? station.departures / station.totalTraffic : 0.5
						)}`}
						class={station?.Number === selectedStation?.Number ? "selected" : ""}
						on:mousedown={() =>
							(selectedStation =
								selectedStation?.Number !== station?.Number ? station : null)}
					>
						<title>
							{station.totalTraffic} trips ({station.departures} departures, {station.arrivals}
							arrivals)
						</title>
					</circle>
				{/each}
			{/key}
		{/if}
	</svg>
</div>

<div class="legend">
	<div style="--departure-ratio: 1">More departures</div>
	<div style="--departure-ratio: 0.5">Balanced</div>
	<div style="--departure-ratio: 0">More arrivals</div>
</div>

<style>
	@import url("$lib/global.css");

	header {
		display: flex;
		gap: 1em;
		align-items: baseline;
	}

	label {
		margin-left: auto;
	}

	time,
	em {
		display: block;
	}

	em {
		color: #666;
		font-style: italic;
	}

	#map {
		flex: 1;
		position: relative;
		min-height: 70vh;
	}

	#map svg {
		position: absolute;
		inset: 0;
		width: 100%;
		height: 100%;
		z-index: 1;
		pointer-events: none;
	}

	#map svg path {
		pointer-events: auto;
	}

	#map svg circle,
	.legend > div {
		--color-departures: steelblue;
		--color-arrivals: darkorange;
		--color: color-mix(
			in oklch,
			var(--color-departures) calc(100% * var(--departure-ratio)),
			var(--color-arrivals)
		);
	}

	#map svg circle {
		fill: var(--color);
		fill-opacity: 0.6;
		stroke: white;
		stroke-width: 1;
		pointer-events: auto;
		transition: opacity 0.2s ease;
	}

	#map svg:has(circle.selected) circle:not(.selected) {
		opacity: 0.3;
	}

	.legend {
		display: flex;
		gap: 1px;
		margin-block: 1em;
	}

	.legend > div {
		flex: 1;
		background: var(--color);
		padding: 0.5em 1.5em;
		color: white;
	}

	.legend > div:nth-child(1) {
		text-align: left;
	}

	.legend > div:nth-child(2) {
		text-align: center;
	}

	.legend > div:nth-child(3) {
		text-align: right;
	}
</style>