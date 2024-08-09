<script>
    import { onMount } from "svelte";
    import maplibregl from "maplibre-gl";
    import "maplibre-gl/dist/maplibre-gl.css";
    import * as pmtiles from "pmtiles";
    import BaseLayer from "/src/data/toronto.json";

    let map;
    let scale = new maplibregl.ScaleControl({
        maxWidth: 100,
        unit: "metric",
    });
    let PMTILES_URL = "/src/data/toronto.pmtiles";

    const maxBounds = [
        [-79.7712, 43.44], // SW coords
        [-78.914763, 43.93074], // NE coords
    ];

    onMount(async () => {
        let protocol = new pmtiles.Protocol();
        maplibregl.addProtocol("pmtiles", protocol.tile);

        map = new maplibregl.Map({
            container: "map",
            style: {
                version: 8,
                name: "Empty",
                glyphs: "https://schoolofcities.github.io/fonts/fonts/{fontstack}/{range}.pbf",
                sources: {},
                layers: [
                    {
                        id: "background",
                        type: "background",
                        paint: {
                            "background-color": "rgba(0,0,0,0)",
                        },
                    },
                ],
            },
            center: [-79.405, 43.647],
            zoom: 9.4,
            maxZoom: 16.5,
            minZoom: 9,
            maxPitch: 85,
            bearing: -17.1,
            projection: "globe",
            scrollZoom: true,
            maxBounds: maxBounds,
            attributionControl: true,
        });

        const attributions = [
            '<a href="https://openstreetmap.org">OpenStreetMap</a>',
            '<a href="https://open.toronto.ca/">City of Toronto </a>',
        ];
        const attributionString = attributions.join(", ");
        let protoLayers = BaseLayer;

        map.addControl(scale, "bottom-left");
        map.addControl(new maplibregl.NavigationControl(), "top-left");
        map.dragRotate.disable();

        const response = await fetch("/src/data/campus_locations.geojson");
        const geojson = await response.json();

        map.on("load", function () {
            map.addSource("protomaps", {
                type: "vector",
                url: "pmtiles://" + PMTILES_URL,
                attribution: attributionString,
                attributionControl: false,
            });

            protoLayers.forEach((e) => {
                map.addLayer(e);
            });

            map.addSource("campus-locations", {
                type: "geojson",
                data: geojson,
            });

            map.addLayer({
                id: "campus-points",
                type: "circle",
                source: "campus-locations",
                paint: {
                    "circle-radius": 8,
                    "circle-color": "#007cbf",
                },
            });
        });

        // Console log zoom and center
        map.on("moveend", () => {
            console.log("Map center:", map.getCenter());
            console.log("Map zoom level:", map.getZoom());
        });
    });
</script>

<div id="map"></div>

<style>
    #map {
        width: 720px;
        height: 50vh;
    }
</style>
