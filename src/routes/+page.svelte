<script>
    import mapboxgl from 'mapbox-gl';
    import '../../node_modules/mapbox-gl/dist/mapbox-gl.css';
    import { onMount } from 'svelte';
    import * as d3 from 'd3';

    mapboxgl.accessToken = 'pk.eyJ1IjoibnNob3I0NyIsImEiOiJjbTNqZHpkemowMmNyMnFwdGRib3hrdHFrIn0.JyaBZZ6ZeBmeptjJhSR8nQ';

    let map;
    let stations = [];
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
                data: 'https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson', // Replace with actual URL
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

            const url = 'https://vis-society.github.io/labs/8/data/bluebikes-stations.csv';
            stations = await d3.csv(url);
            console.log('Stations data loaded:', stations);

            $: map?.on('move', () => mapViewChanged++);

        } catch (error) {
            console.error('Error initializing map or fetching data:', error);
        }
    });
</script>

<h2>Boston and Cambridge Bike Paths</h2>
<div id="map">
    <div id="svg-map">
        {#key mapViewChanged}
            <svg>
                {#each stations as station}
                    <circle {...getCoords(station)} r="3" fill="steelblue" />
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
</style>