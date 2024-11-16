<script>
    import mapboxgl from 'mapbox-gl';
    import '../../node_modules/mapbox-gl/dist/mapbox-gl.css';
    mapboxgl.accessToken = 'pk.eyJ1IjoibnNob3I0NyIsImEiOiJjbTNqZHpkemowMmNyMnFwdGRib3hrdHFrIn0.JyaBZZ6ZeBmeptjJhSR8nQ';
    import { onMount } from 'svelte';

    onMount(() => {
    let map = new mapboxgl.Map({
        container: 'map',
        style: 'mapbox://styles/mapbox/streets-v12',
        center: [-71.0926, 42.3601],
        zoom: 12,
        minZoom: 5,
        maxZoom: 18
    });
    });
</script>

<style>
@import url('$lib/global.css');

</style>

<body>Boston Bike Paths</body>
<div id="map" style="width: 100%; height: 500px;"></div>

