<script>
    import { onMount } from "svelte";
    import maplibregl from "maplibre-gl";
    import "maplibre-gl/dist/maplibre-gl.css";
  
    let map;
  
    onMount(() => {
      map = new maplibregl.Map({
        container: "map", // The id of the div where the map will be displayed
        style: "https://basemaps.cartocdn.com/gl/positron-gl-style/style.json", // Map style URL
        center: [-79.38919989374392, 43.641655778152966], // Set to UofT or desired center
        zoom: 12 // Initial zoom level
      });
  
      // Add any additional map layers, controls, or interactions here
    });
  </script>
  
  <style>
    #map {
      width: 720px;
      height: 50vh;
    }
  </style>
  
  <div id="map"></div>
  