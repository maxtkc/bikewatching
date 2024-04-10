<script>
import mapboxgl from "mapbox-gl";
import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
import { onMount } from "svelte";
import * as d3 from "d3";

mapboxgl.accessToken = "pk.eyJ1IjoibWF4dGtjIiwiYSI6ImNsaWJybGQ1YTBkejUzZW5iYm5iZXp6eW8ifQ.CHNbK4sRwKNVS08ArP-kXw";

    let stations = [];
    let departures;
    let arrivals;
    let filteredStations = [];
    let filteredDepartures;
    let filteredArrivals;
    let map;
    let mapViewChanged = 0;
    let timeFilter = -1;
    let trips = [];
    let filteredTrips = [];

    $: map?.on("move", evt => mapViewChanged++);

    function getCoords (station) {
	let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
	let {x, y} = map.project(point);
	return {cx: x, cy: y};
    }

    onMount(async () => {
        map = new mapboxgl.Map({
            center: [-71.13184774601001, 42.36481600519267],
            style: "mapbox://styles/mapbox/streets-v12",
            container: "map",
            zoom: 12,
        });

        stations = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-stations.csv");

        // const trips = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv");
        trips = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv").then(trips => {
            for (let trip of trips) {
                trip.started_at = new Date(trip.started_at);
                trip.ended_at = new Date(trip.ended_at);
            }
            return trips;
        });

        departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
        arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);

        stations = stations.map(station => {
            let id = station.Number;
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;
            return station;
        });

        await new Promise(resolve => map.on("load", resolve));

        map.addSource("boston_route", {
            type: "geojson",
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
        });
        map.addSource("cambridge_route", {
            type: "geojson",
            data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
        });
        map.addLayer({
            id: "boston_lane_layer", // A name for our layer (up to you)
            type: "line", // one of the supported layer types, e.g. line, circle, etc.
            source: "boston_route", // The id we specified in `addSource()`
            paint: {
                // paint params, e.g. colors, thickness, etc.
                "line-color": "green",
                "line-width": 2,
                "line-opacity": .5,
            },
        });
        map.addLayer({
            id: "cambridge_lane_layer", // A name for our layer (up to you)
            type: "line", // one of the supported layer types, e.g. line, circle, etc.
            source: "cambridge_route", // The id we specified in `addSource()`
            paint: {
                // paint params, e.g. colors, thickness, etc.
                "line-color": "green",
                "line-width": 2,
                "line-opacity": .5,
            },
        });
    })

    function minutesSinceMidnight (date) {
        return date.getHours() * 60 + date.getMinutes();
    }

    let stationFlow = d3.scaleQuantize()
	.domain([0, 1])
	.range([0, 0.5, 1]);

    $: radiusScale = d3.scaleSqrt()
	.domain([0, d3.max(stations, d => d.totalTraffic)])
	.range([0, 25]);

    $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
                     .toLocaleString("en", {timeStyle: "short"});

    $: filteredTrips = timeFilter === -1? trips : trips.filter(trip => {
        let startedMinutes = minutesSinceMidnight(trip.started_at);
        let endedMinutes = minutesSinceMidnight(trip.ended_at);
        return Math.abs(startedMinutes - timeFilter) <= 60
            || Math.abs(endedMinutes - timeFilter) <= 60;
    });

    $: {
        filteredDepartures = d3.rollup(filteredTrips, v => v.length, d => d.start_station_id);
        filteredArrivals = d3.rollup(filteredTrips, v => v.length, d => d.end_station_id);

        filteredStations = stations.map(station => {
            station = {...station};
            let id = station.Number;
            station.arrivals = filteredArrivals.get(id) ?? 0;
            station.departures = filteredDepartures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;
            return station;
        });
    }

</script>

<h1>BikeWatching</h1>

<header>
    <label> Filter by time:
        <input type=range min=-1 max=1440 bind:value={timeFilter} />
    {#if timeFilter === -1}
    <em>(any time)</em>
    {:else}
    <time>{timeFilterLabel}</time>
    {/if}
    </label>
</header>

<div id="map">
    <svg>
        {#key mapViewChanged}
        {#each filteredStations as station }
        <circle { ...getCoords(station) } r={radiusScale(station.totalTraffic)} fill="steelblue" style="--departure-ratio: { stationFlow(station.departures / station.totalTraffic) }"/>
        {/each}
        {/key}
    </svg>
</div>

<div class="legend">
	<div style="--departure-ratio: 1">More departures</div>
	<div style="--departure-ratio: 0.5">Balanced</div>
	<div style="--departure-ratio: 0">More arrivals</div>
</div>

<style>
    @import url("$lib/global.css");
    #map {
        flex: 1;
        background-color: red;
    }
    #map svg {
        position: absolute;
        z-index: 1;
        width: 100%;
        height: 100%;
        pointer-events: none;
    }
    #map svg circle {
        fill-opacity: .6;
        stroke: white;
        fill: var(--color);
    }
    header {
        display: flex;
        gap: 1em;
        align-items: baseline;
        margin-left: auto;
    }
    header em, header time {
        display: block
    }

    #map circle, .legend > div {
        --color-departures: steelblue;
        --color-arrivals: darkorange;
        --color: color-mix(
        in oklch,
        var(--color-departures) calc(100% * var(--departure-ratio)),
        var(--color-arrivals)
        );
    }
    .legend {
        display: flex;
        margin-block: 20px, 20px;
    }
    .legend > div {
        background-color: var(--color);
        flex: 1;
        text-align: center;
        color: white;
    }
</style>
