<script>
	import { onMount } from "svelte";
	import maplibregl from "maplibre-gl";
	import "maplibre-gl/dist/maplibre-gl.css";
	import * as pmtiles from "pmtiles";
	import BaseLayer from "../data/toronto.json";
	import CampusLocations from "../data/campus_locations.geo.json";
	import "../assets/global.css";

	let map;
	let scale = new maplibregl.ScaleControl({
		maxWidth: 100,
		unit: "metric",
	});

	let PMTILES_URL = "/toronto.pmtiles";

	const campusBounds = [
		[-79.70, 43.60], // SW corner
		[-79.15, 43.74], // NE corner
	];

	const padding = 50;
	const bearing = -17.1;

	let defaultColor = "#1E3765";
	let highlightColor = "#AB1368";

	onMount(async () => {
		let protocol = new pmtiles.Protocol();
		maplibregl.addProtocol("pmtiles", protocol.tile);

		// const rootStyles = getComputedStyle(document.documentElement);
		// defaultColor = rootStyles.getPropertyValue("--brandGray").trim();
		// highlightColor = rootStyles.getPropertyValue("--brandMedBlue").trim();

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
			bounds: campusBounds,
			fitBoundsOptions: {
				padding: padding,
				bearing: bearing,
			},
			maxPitch: 0,
			maxZoom: 11,
			projection: "globe",
			scrollZoom: false,
			dragPan: false,
			dragRotate: false,
			doubleClickZoom: false,
			touchZoomRotate: false,
			keyboard: false,
			interactive: false,
			attributionControl: false,
		});

		const attributions = [
			'<a href="https://openstreetmap.org">OpenStreetMap</a>',
			'<a href="https://open.toronto.ca/">City of Toronto</a>',
		];
		const attributionString = attributions.join(", ");

		map.addControl(
			new maplibregl.AttributionControl({
				compact: true,
			}),
			"bottom-right",
		);

		// Make the attribution control compact by default
		document.querySelector(".maplibregl-ctrl-attrib").classList.add("maplibregl-compact");

		map.addControl(scale, "bottom-left");

		map.on("load", () => {
			map.addSource("protomaps", {
				type: "vector",
				url: "pmtiles://" + PMTILES_URL,
				attribution: attributionString,
				attributionControl: false,
			});

			BaseLayer.forEach((layer) => {
				map.addLayer(layer);
			});

			map.addSource("campus-locations", {
				type: "geojson",
				data: CampusLocations,
			});

			map.addLayer({
				id: "campus-points",
				type: "circle",
				source: "campus-locations",
				paint: {
					"circle-radius": 8,
					"circle-color": defaultColor,
				},
			});

			map.addLayer({
				id: "campus-points-label",
				type: "symbol",
				source: "campus-locations",
				minzoom: 8,
				layout: {
					"text-field": ["get", "Campus"],
					"text-size": 18,
					"text-anchor": "top",
					"text-offset": [0, -1.7],
					"text-font": ["TradeGothic LT Bold"],
				},
				paint: {
					"text-color": "#1E3765",
					"text-halo-color": "#FFFFFF",
					"text-halo-width": 2,
				},
			});

			const popup = new maplibregl.Popup({
				closeButton: false,
				closeOnClick: false,
			});

			map.on("mouseenter", "campus-points", (e) => {
				map.getCanvas().style.cursor = "pointer";
				const coordinates = e.features[0].geometry.coordinates.slice();
				const { Campus, Address } = e.features[0].properties;
				map.setPaintProperty("campus-points", "circle-color", [
					"match",
					["get", "Campus"],
					Campus,
					highlightColor,
					defaultColor,
				]);
				map.setPaintProperty("campus-points-label", "text-color", [
					"match",
					["get", "Campus"],
					Campus,
					highlightColor,
					defaultColor,
				]);
			});

			map.on("mouseleave", "campus-points", () => {
				map.getCanvas().style.cursor = "";
				popup.remove();
				map.setPaintProperty(
					"campus-points",
					"circle-color",
					defaultColor,
				);
				map.setPaintProperty(
					"campus-points-label",
					"text-color",
					defaultColor,
				);
			});

			map.on("click", "campus-points", (e) => {
				const { URL } = e.features[0].properties;
				if (URL) {
					window.open(URL, "_blank");
				}
			});

			// essentially duplicated this code, but for the label, maybe there is a more efficient way?

			map.on("mouseenter", "campus-points-label", (e) => {
				map.getCanvas().style.cursor = "pointer";
				const coordinates = e.features[0].geometry.coordinates.slice();
				const { Campus, Address } = e.features[0].properties;
				map.setPaintProperty("campus-points", "circle-color", [
					"match",
					["get", "Campus"],
					Campus,
					highlightColor,
					defaultColor,
				]);
				map.setPaintProperty("campus-points-label", "text-color", [
					"match",
					["get", "Campus"],
					Campus,
					highlightColor,
					defaultColor,
				]);
			});

			map.on("mouseleave", "campus-points-label", () => {
				map.getCanvas().style.cursor = "";
				popup.remove();
				map.setPaintProperty(
					"campus-points",
					"circle-color",
					defaultColor,
				);
				map.setPaintProperty(
					"campus-points-label",
					"text-color",
					defaultColor,
				);
			});

			map.on("click", "campus-points-label", (e) => {
				const { URL } = e.features[0].properties;
				if (URL) {
					window.open(URL, "_blank");
				}
			});
		});

		window.addEventListener("resize", () => {
			map.resize();
			map.fitBounds(campusBounds, {
				padding: padding,
				duration: 0,
				bearing: bearing,
			});
		});
	});
</script>

<div id="map"></div>

<style>
	#map {
		width: 100vw;
		height: 100vh;
	}
</style>
