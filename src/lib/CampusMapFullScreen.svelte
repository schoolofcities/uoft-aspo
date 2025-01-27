<script>
	import { onMount } from "svelte";
	import maplibregl from "maplibre-gl";
	import "maplibre-gl/dist/maplibre-gl.css";
	import * as pmtiles from "pmtiles";
	import BaseLayer from "../data/toronto.json";
	// import BaseLayerYellow from "../data/toronto-yellow.json";
	// import BaseLayerDark from "../data/toronto-dark.json";
	// import BaseLayerNavy from "../data/toronto-navy.json";
	import BaseLayerGreen from "../data/toronto-green.json";
	// import BaseLayerWhiteBlue from "../data/toronto-whiteblue.json";
	import CampusLocations from "../data/campus_locations.geo.json";
	import "../assets/global.css";

	import logo from "../assets/aspo-logo-blue.png";

	// export let mapStyle;

	// let base = [];
	// if (mapStyle === "dark") {
	// 	base = BaseLayerDark;
	// } else if (mapStyle === "yellow") {
	// 	base = BaseLayerYellow;
	// } else if (mapStyle === "navy") {
	// 	base = BaseLayerNavy;
	// } else if (mapStyle === "green") {
	// 	base = BaseLayerGreen;
	// } else if (mapStyle === "whiteblue") {
	// 	base = BaseLayerWhiteBlue;
	// } else {
	// 	base = BaseLayer;
	// }

	let base = BaseLayerGreen;

	let map;
	let scale = new maplibregl.ScaleControl({
		maxWidth: 100,
		unit: "metric",
	});

	let PMTILES_URL = "/uoft-aspo/toronto.pmtiles";
	// let PMTILES_URL = 'https://api.protomaps.com/tiles/v4.json?key=7f48bb9c6a1f1e3b'

	const campusBounds = [
		[-79.70, 43.60], // SW corner
		[-79.15, 43.74], // NE corner
	];

	const padding = 50;
	const bearing = -17.1;

	let defaultColor = "#1E3765"; // "#1E3765";
	let highlightColor = "#AB1368";
	let haloColor = "#ffffff"; //"#fff";

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
			'ASPO gratefully acknowledges Jeff Allen and Scott McCallum at the <a href="https://schoolofcities.utoronto.ca/" target= "_blank" style="color: black; text-decoration: underline;">School of Cities</a> for designing this map. Data for the map are from <a href="https://openstreetmap.org" target= "_blank" style="color: black; text-decoration: underline;"> OpenStreetMap</a>'
		];
		const attributionString = attributions.join(", ");

		map.addControl(
		new maplibregl.AttributionControl({
			compact: false, // Ensures the attribution is in compact mode
		}),
			'bottom-right'
		);

		map.addControl(scale, "bottom-right");

		map.on("load", () => {
			map.addSource("protomaps", {
				type: "vector",
				url: "pmtiles://" + PMTILES_URL,
				attribution: attributionString,
				attributionControl: false,
			});

			base.forEach((layer) => {
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
					"circle-stroke-color": haloColor,
           			"circle-stroke-width": 2,
					'circle-stroke-opacity': 0.7
				},
			});

			map.addLayer({
				id: "campus-points-label",
				type: "symbol",
				source: "campus-locations",
				minzoom: 8,
				layout: {
					'text-field': ['get', 'Campus'], 
					'text-size': 19,
					'text-anchor': 'top', 
					'text-offset': [0, -1.7],
					"text-font": [
						"TradeGothic LT Bold"
					],
				},
				paint: {
					'text-color': defaultColor, 
					'text-halo-color': haloColor, 
					'text-halo-width': 2, 
					'text-halo-blur': 1
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

<div class="overlay">
	<a href="https://www.aspo.utoronto.ca" target="_blank" rel="noopener noreferrer">
		<img src={logo} alt="Overlay Image">
	</a>
</div>

<style>

	#map {
		width: 100vw;
		height: 100vh;
		position: relative;
	}

	.overlay {
		position: absolute; 
		top: 0; 
		left: 0; 
		width: 275px; 
		height: 37px; 
		background-color: rgba(255, 255, 255, 0.9);
		z-index: 10; 
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 10px;
	}

	.overlay a {
		display: block;
		width: 100%;
		height: 100%;
		text-align: center;
	}

	.overlay img {
		margin-left: 10px;
		margin-right: 10px;
		max-width: 100%;
		max-height: 100%;
	}

	.overlay img:hover {
		opacity: 0.8;
	}

</style>
