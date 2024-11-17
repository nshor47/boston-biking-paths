<script>
    import mapboxgl from 'mapbox-gl';
    import '../../node_modules/mapbox-gl/dist/mapbox-gl.css';
    import { onMount } from 'svelte';
    import * as d3 from 'd3';

    mapboxgl.accessToken = 'pk.eyJ1IjoibnNob3I0NyIsImEiOiJjbTNqZHpkemowMmNyMnFwdGRib3hrdHFrIn0.JyaBZZ6ZeBmeptjJhSR8nQ';

    let map;
    let stations = [];
    let trips = [];
    let mapViewChanged = 0;

    let timeFilter = -1; // Slider value, -1 means no filtering
    $: timeFilterLabel = timeFilter === -1
        ? "any time"
        : new Date(0, 0, 0, 0, timeFilter).toLocaleString('en', { timeStyle: 'short' });

    // Function to get the coordinates of the stations
    function getCoords(station) {
        if (!map) return { cx: 0, cy: 0 };
        const { x, y } = map.project([+station.Long, +station.Lat]);
        return { cx: x, cy: y };
    }

    // Function to convert date to minutes since midnight
    function minutesSinceMidnight(date) {
        return date.getHours() * 60 + date.getMinutes();
    }

    onMount(async () => {
        try {
            // Initialize Mapbox map
            map = new mapboxgl.Map({
                container: 'map',
                style: 'mapbox://styles/mapbox/streets-v12',
                center: [-71.0926, 42.3601],
                zoom: 12,
            });

            await new Promise((resolve) => map.on('load', resolve));

            // Add Boston and Cambridge bike route layers to map
            map.addSource('boston_route', {
                type: 'geojson',
                data: 'https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D',
            });

            map.addLayer({
                id: 'boston-bike-routes',
                type: 'line',
                source: 'boston_route',
                layout: {
                    'line-join': 'round',
                    'line-cap': 'round',
                },
                paint: {
                    'line-color': 'green',
                    'line-width': 2,
                    'line-opacity': 0.5,
                },
            });

            map.addSource('cambridge_route', {
                type: 'geojson',
                data: 'https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson',
            });

            map.addLayer({
                id: 'cambridge-bike-routes',
                type: 'line',
                source: 'cambridge_route',
                layout: {
                    'line-join': 'round',
                    'line-cap': 'round',
                },
                paint: {
                    'line-color': 'green',
                    'line-width': 2,
                    'line-opacity': 0.5,
                },
            });

            // Fetch data for stations and trips
            const stationsUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-stations.csv';
            stations = await d3.csv(stationsUrl);

            const tripsUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv';
            trips = await d3.csv(tripsUrl);

            // Convert trip start and end times to Date objects
            trips = await d3.csv(tripsUrl).then((trips) => {
                for (let trip of trips) {
                    trip.started_at = new Date(trip.started_at);
                    trip.ended_at = new Date(trip.ended_at);
                }
                return trips;
            });

            // Create rolls for arrivals and departures by station
            const departures = d3.rollup(trips, (v) => v.length, (d) => d.start_station_id);
            const arrivals = d3.rollup(trips, (v) => v.length, (d) => d.end_station_id);

            // Update stations with arrival and departure counts
            stations = stations.map((station) => {
                let id = station.Number;
                station.arrivals = arrivals.get(id) ?? 0;
                station.departures = departures.get(id) ?? 0;
                station.totalTraffic = station.arrivals + station.departures;
                return station;
            });

            $: map?.on('move', () => mapViewChanged++);
        } catch (error) {
            console.error('Error initializing map or fetching data:', error);
        }
    });

    // Conditional scale for radius size of stations
    $: radiusScale = d3
        .scaleSqrt()
        .domain([0, d3.max(stations, (d) => d.totalTraffic)])
        .range(timeFilter === -1 ? [0, 25] : [3, 50]);

    // Filter trips based on timeFilter
    $: filteredTrips =
        timeFilter === -1
            ? trips
            : trips.filter((trip) => {
                let startedMinutes = minutesSinceMidnight(trip.started_at);
                let endedMinutes = minutesSinceMidnight(trip.ended_at);
                return (
                    Math.abs(startedMinutes - timeFilter) <= 60 ||
                    Math.abs(endedMinutes - timeFilter) <= 60
                );
            });

    // Filter arrivals and departures based on filtered trips
    $: filteredArrivals = d3.rollup(filteredTrips, (v) => v.length, (d) => d.start_station_id);
    $: filteredDepartures = d3.rollup(filteredTrips, (v) => v.length, (d) => d.end_station_id);

    // Filter stations based on filtered trips
    $: filteredStations = stations.map((station) => {
        // Clone station object to avoid mutation
        station = { ...station };

        // Update station data based on filtered arrivals and departures
        station.arrivals = filteredArrivals.get(station.Number) ?? 0;
        station.departures = filteredDepartures.get(station.Number) ?? 0;
        station.totalTraffic = station.arrivals + station.departures;

        return station;
    });
</script>

<header>
    <h2>Boston and Cambridge Bike Paths</h2>
    <label>
        Filter by time:
        <input type="range" min="-1" max="1440" bind:value={timeFilter} />
        {#if timeFilter === -1}
            <em>any time</em>
        {:else}
            <time>{timeFilterLabel}</time>
        {/if}
    </label>
</header>

<div id="map">
    <div id="svg-map">
        {#key mapViewChanged}
            <svg>
                {#each filteredStations as station}
                    <circle
                        {...getCoords(station)}
                        r={radiusScale(station.totalTraffic)}
                        fill="steelblue"
                        fill-opacity="0.6"
                        stroke="white"
                    />
                {/each}
            </svg>
        {/key}
    </div>
</div>

<style>
    header {
        display: flex;
        gap: 1em;
        align-items: baseline;
        margin-bottom: 1em;
    }

    header label {
        margin-left: auto;
    }

    header time, header em {
        display: block;
    }

    header em {
        color: gray;
        font-style: italic;
    }

    #map {
        position: relative;
        width: 100%;
        height: 500px;
        margin-top: 10px;
    }

    #svg-map {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        pointer-events: none;
        z-index: 10;
    }

    #svg-map svg {
        width: 100%;
        height: 100%;
    }

    #svg-map circle {
        stroke: white;
        fill-opacity: 0.6;
    }
</style>