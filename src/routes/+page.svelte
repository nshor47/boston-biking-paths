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

    function getCoords(station) {
        if (!map) return { cx: 0, cy: 0 };
        const { x, y } = map.project([+station.Long, +station.Lat]);
        return { cx: x, cy: y };
    }

    onMount(async () => {
        try {
            map = new mapboxgl.Map({
                container: 'map',
                style: 'mapbox://styles/mapbox/streets-v12',
                center: [-71.0926, 42.3601],
                zoom: 12,
            });

            await new Promise((resolve) => map.on('load', resolve));

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

            const stationsUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-stations.csv';
            stations = await d3.csv(stationsUrl);

            const tripsUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv';
            trips = await d3.csv(tripsUrl);

            // Calculate arrivals, departures, and total traffic
            const departures = d3.rollup(trips, (v) => v.length, (d) => d.start_station_id);
            const arrivals = d3.rollup(trips, (v) => v.length, (d) => d.end_station_id);

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

    // Define the square root scale for circle radius
    $: radiusScale = d3
        .scaleSqrt()
        .domain([0, d3.max(stations, (d) => d.totalTraffic)])
        .range([0, 25]);
</script>

<h2>Boston and Cambridge Bike Paths</h2>
<div id="map">
    <div id="svg-map">
        {#key mapViewChanged}
            <svg>
                {#each stations as station}
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