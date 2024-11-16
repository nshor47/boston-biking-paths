<script>
    import mapboxgl from 'mapbox-gl';
    import '../../node_modules/mapbox-gl/dist/mapbox-gl.css';
    import { onMount } from 'svelte';
    import * as d3 from 'd3';

    mapboxgl.accessToken = 'pk.eyJ1IjoibnNob3I0NyIsImEiOiJjbTNqZHpkemowMmNyMnFwdGRib3hrdHFrIn0.JyaBZZ6ZeBmeptjJhSR8nQ';

    let map;
    let stations = [];
    let mapViewChanged = 0; // Variable to track map movements

    // Helper function to calculate circle coordinates
    function getCoords(station) {
        if (!map) return { cx: 0, cy: 0 }; // Return dummy values if map is undefined
        const { x, y } = map.project([+station.Long, +station.Lat]);
        return { cx: x, cy: y };
    }

    onMount(async () => {
        try {
            // Initialize the Mapbox map
            map = new mapboxgl.Map({
                container: 'map',
                style: 'mapbox://styles/mapbox/streets-v12',
                center: [-71.0926, 42.3601],
                zoom: 12,
            });

            // Wait for the map to load
            await new Promise((resolve) => map.on('load', resolve));

            // Add Boston bike routes
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

            // Fetch station data and log to console
            const url = 'https://vis-society.github.io/labs/8/data/bluebikes-stations.csv';
            stations = await d3.csv(url);
            console.log('Stations data loaded:', stations);

            // Trigger Svelte to update the DOM when the map moves
            $: map?.on('move', () => mapViewChanged++);

        } catch (error) {
            console.error('Error initializing map or fetching data:', error);
        }
    });
</script>
<h1>Boston and Cambridge Bike Paths</h1> <!-- Added the missing title -->
<div id="map">
    <div id="svg-map">
        <!-- Overlay SVG for circles -->
        {#key mapViewChanged}
            <svg>
                {#each stations as station}
                    <circle {...getCoords(station)} r="5" fill="steelblue" />
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
    margin-top: 0px; /* Add margin to push the map down */
}

#svg-map {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none; /* Allow interaction with the map */
    z-index: 10; /* Keep SVG above the map */
}

    #svg-map svg {
        width: 100%;
        height: 100%;
    }
    
</style>