<script>
    import mapboxgl from 'mapbox-gl';
    import '../../node_modules/mapbox-gl/dist/mapbox-gl.css';
    import { onMount, onDestroy } from 'svelte';
  
    mapboxgl.accessToken = 'pk.eyJ1IjoibnNob3I0NyIsImEiOiJjbTNqZHpkemowMmNyMnFwdGRib3hrdHFrIn0.JyaBZZ6ZeBmeptjJhSR8nQ';
  
    let map;
  
    onMount(async () => {
      if (typeof window !== 'undefined') {
        const container = document.getElementById('map');
        if (container.firstChild) {
          container.innerHTML = ''; 
        }
  
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
      }
    });
  
    onDestroy(() => {
      if (map) {
        map.remove();
      }
    });
  </script>
  
  <style>
    #map {
      width: 100%;
      height: 500px;
    }
  </style>
  
  <div class="container">
    <h1>Boston and Cambridge Bike Paths</h1>
    <div id="map"></div>
  </div>