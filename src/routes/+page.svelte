

<script>
    import mapboxgl from "mapbox-gl";
    import * as d3 from 'd3';
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    mapboxgl.accessToken = "pk.eyJ1IjoibW9yYWxlc2RhbmllbGEiLCJhIjoiY2x1dDBybXhiMGhzaDJqbzlxdXpybXp2NCJ9.1-e16wW3DpdZIxfz5rw14g";
    import { onMount } from "svelte";
    let map;
    let stations = [];
    let trips = [];
    let departures = [];
    let arrivals = [];
    let mapViewChanged = 0;
    let timeFilter = -1;
    let filteredTrips = [];
    let departuresByMinute = Array.from({ length: 1440 }, () => []);
    let arrivalsByMinute = Array.from({ length: 1440 }, () => []);
    let stationFlow = d3.scaleQuantize()
        .domain([0, 1])
        .range([0, 0.5, 1]);
    onMount(async () => {
        map = new mapboxgl.Map({
            container: 'map',
            center: [-71.09451, 42.36027],
            zoom: 13,
            style: 'mapbox://styles/mapbox/streets-v11',
        });
        await new Promise(resolve => map.on("load", resolve));
        map.addSource("bostonRoute", {
            type: "geojson",
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
        });
        map.addSource("cambridgeRoute", {
            type: "geojson",
            data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
        });
        map.addLayer({
            id: "bostonBikeLanes", // A name for our layer (up to you)
            type: "line", // one of the supported layer types, e.g. line, circle, etc.
            source: "bostonRoute", // The id we specified in `addSource()`
            paint: {
                "line-color": "green", // Set the line color to a nice green
                "line-width": 3, // Set the line thickness to 10
                "line-opacity":.40
            },
        });
        map.addLayer({
            id: "cambridgeBikeLanes", // A name for our layer (up to you)
            type: "line", // one of the supported layer types, e.g. line, circle, etc.
            source: "cambridgeRoute", // The id we specified in `addSource()`
            paint: {
                "line-color": "green", // Set the line color to a nice green
                "line-width": 3, // Set the line thickness to 10
                "line-opacity": .40
            },
        });
        stations = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-stations.csv");
        let TRIP_DATA_URL = "https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv";
        trips = await d3.csv(TRIP_DATA_URL).then(trips => {
            for (let trip of trips) {
                trip.started_at = new Date(trip.started_at);
                trip.ended_at = new Date(trip.ended_at);
                let startMinute = minutesSinceMidnight(trip.started_at);
                let endMinute = minutesSinceMidnight(trip.ended_at);
                departuresByMinute[startMinute].push(trip);
                arrivalsByMinute[endMinute].push(trip);
            }
            return trips;
        });
        arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);
        departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
        stations = stations.map(station => {
            let id = station.Number;
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;
            return station;
        });
    })
    function getCoords (station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let {x, y} = map.project(point);
        return {cx: x, cy: y};
    }
    $: radiusScale = d3.scaleSqrt()
        .domain([0, d3.max(stations, d => d.totalTraffic)])
        .range([0, 25]);
    $: map?.on("move", evt => mapViewChanged++);
    $: timeFilterLabel = timeFilter === -1 ? '(any time)' : new Date(0, 0, 1, 0, timeFilter)
        .toLocaleString("en", {timeStyle: "short"});
    function minutesSinceMidnight (date) {
        return date.getHours() * 60 + date.getMinutes();
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
    $: filteredDepartures = filterByMinute(departuresByMinute, timeFilter);
    $: filteredArrivals = filterByMinute(arrivalsByMinute, timeFilter);
    $: filteredDeparturesAggregate = d3.rollup(filteredDepartures, v => v.length, d => d.start_station_id);
    $: filteredArrivalsAggregate = d3.rollup(filteredArrivals, v => v.length, d => d.end_station_id);
    $: stations = timeFilter === -1 ? stations : stations.map(station => {
        station = { ...station }; // Clone to avoid mutating the original
        let id = station.Number;
        station.departures = filteredDeparturesAggregate.get(id) ?? 0;
        station.arrivals = filteredArrivalsAggregate.get(id) ?? 0;
        station.totalTraffic = station.departures + station.arrivals;
        return station;
    });
</script>
<h1>Boston Bike Project</h1>
<p>This interactive map of Boston shows bike traffic times during different hours of the day. </p>
<header class="timescrubb">
    Filter by time:
    <input type="range" min="-1" max="1440" bind:value={timeFilter}>
    <time>
        { timeFilterLabel }
    </time>
</header>
<br>
<div id="map">
    <svg>
        {#key mapViewChanged}
            {#each stations as station}
                <circle
                    { ...getCoords(station) }
                    r={ radiusScale(station.totalTraffic) }
                    style="--departure-ratio: { stationFlow(station.departures / station.totalTraffic) }"
                >
                    <title>
                        {station.totalTraffic} trips ({station.departures} departures, { station.arrivals} arrivals)
                    </title>
                </circle>
            {/each}
        {/key}
    </svg>
</div>
<div class="legend">
    Legend:
    <div style="--departure-ratio: 1">More departures</div>
    <div style="--departure-ratio: 0.5">Balanced</div>
    <div style="--departure-ratio: 0">More arrivals</div>
</div>
<style>
    @import url("$lib/global.css");
    :root {
        --color-departures: steelblue;
        --color-arrivals: darkorange;
    }
    #map {
        flex: 1;
    }
    #map svg {
        width: 100%;
        height: 100%;
        pointer-events: none;
        position: absolute;
        z-index: 1;
        circle {
            pointer-events: auto;
            fill-opacity: 60%;
            stroke: white;
            --color: color-mix(
                    in oklch,
                    var(--color-departures) calc(100% * var(--departure-ratio)),
                    var(--color-arrivals)
            );
            fill: var(--color);
        }
    }
    .timescrubb {
        display: flex;
        gap: 1em;
        align-items: center;
        label {
            margin-left: auto;
        }
    }
    .legend > div {
        display: flex;
        align-items: center;
        gap: 0.5em;
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
        flex-direction: row;
        justify-content: center;
        gap: 1em; /* Adjust based on your design */
        margin-block-start: 1rem;
        margin-block-end: 1rem;
        margin-block: 1em; /* Space above and below the legend */
        align-items: center;
    }
    .legend > div::before {
        content: "";
        display: inline-block;
        width: 20px; /* Swatch size */
        height: 20px; /* Swatch size */
        border-radius: 50%;
    }
    .legend > div:nth-child(1)::before {
        background-color: steelblue;
    }
    .legend > div:nth-child(2)::before {
        background-color: color-mix(
                in oklch,
                var(--color-departures) calc(100% * var(--departure-ratio)),
                var(--color-arrivals)
        );
    }
    .legend > div:nth-child(3)::before {
        background-color: darkorange;
    }
</style>







