<script>
	import { onMount } from "svelte";
	import maplibregl from "maplibre-gl";
	import "maplibre-gl/dist/maplibre-gl.css";
	import * as pmtiles from "pmtiles";
	import { forceSimulation, forceX, forceY, forceCollide } from 'd3-force';

	// import BaseLayer from "../data/toronto.json"; 
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



	const swarmPoints = [];
    CampusLocations.features.forEach(feature => {
		const center = feature.geometry.coordinates;
		const programs = feature.properties.Programs;

		for (let i = 0; i < programs; i++) {
			swarmPoints.push({
				type: 'Feature',
				properties: { ...feature.properties },
				geometry: {
					type: 'Point',
					coordinates: [
						center[0] + (Math.random() - 0.5) * 0.01, // Randomize longitude
						center[1] + (Math.random() - 0.5) * 0.01 // Randomize latitude
					]
				},
				originalCenter: center // Store the original center
			});
		}
	});

	// Function to update the beeswarm layout
    function updateBeeswarm(map) {
        const zoom = map.getZoom();
        const collisionRadius = 0.0062 / Math.pow(2, zoom - 10); // Adjust collision radius based on zoom

		swarmPoints.forEach(point => {
			point.geometry.coordinates = [
				point.originalCenter[0] + (Math.random() - 0.5) * 0.01, // Add slight randomness
				point.originalCenter[1] + (Math.random() - 0.5) * 0.01
			];
		});

        // Flatten the data for the simulation
		const flattenedPoints = swarmPoints.map(point => ({
			...point,
			x: point.geometry.coordinates[0], // Extract longitude
			y: point.geometry.coordinates[1] // Extract latitude
		}));

		// Run the force simulation on the flattened data
		const simulation = forceSimulation(flattenedPoints)
			.force('x', forceX(d => d.x).strength(0.1)) // Attract points to their center longitude
			.force('y', forceY(d => d.y).strength(0.1)) // Attract points to their center latitude
			.force('collide', forceCollide(collisionRadius)) // Prevent overlap (adjust radius based on zoom)
			.stop();

		simulation.tick(100);

		// Map the updated x and y values back to the coordinates
		swarmPoints.forEach((point, i) => {
			point.geometry.coordinates[0] = flattenedPoints[i].x; // Update longitude
			point.geometry.coordinates[1] = flattenedPoints[i].y; // Update latitude
		});

        // Update the GeoJSON with the new positions
        const updatedFeatures = swarmPoints.map(point => ({
            ...point,
            geometry: {
                type: 'Point',
                coordinates: [point.geometry.coordinates[0], point.geometry.coordinates[1]]
            }
        }));

        // Update the map source
        map.getSource('beeswarm-source').setData({
            type: 'FeatureCollection',
            features: updatedFeatures
        });
    }









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
			''
		];
		const attributionString = attributions.join(", ");

		map.addControl(
			new maplibregl.AttributionControl({
				compact: false, // Ensures the attribution is in compact mode
			}),
			'bottom-right'
		);

		// map.addControl(scale, "bottom-left");

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



			map.addSource('beeswarm-source', {
                type: 'geojson',
                data: {
                    type: 'FeatureCollection',
                    features: swarmPoints
                }
            });

            // Add the beeswarm layer
            map.addLayer({
                id: 'beeswarm',
                type: 'circle',
                source: 'beeswarm-source',
                paint: {
                    'circle-radius': 8, // Fixed radius for beeswarm points
                    'circle-color': '#F1C500', // Fill color
                    'circle-opacity': 0.1, // 50% fill opacity
                    'circle-stroke-width': 1, // Stroke width
                    'circle-stroke-color': '#ffffff', // Stroke color
                    'circle-stroke-opacity': 0.1 // Fully opaque stroke
                }
            });

            updateBeeswarm(map);

			// SIMPLE POINTS WHAT WE HAD ORIGINALLY

			// map.addLayer({
			// 	id: "campus-points",
			// 	type: "circle",
			// 	source: "campus-locations",
			// 	paint: {
			// 		"circle-radius": 8,
			// 		"circle-color": defaultColor,
			// 		"circle-stroke-color": haloColor,
           	// 		"circle-stroke-width": 2,
			// 		'circle-stroke-opacity': 0.7
			// 	},
			// });

			// map.addLayer({
			// 	id: "campus-points-label",
			// 	type: "symbol",
			// 	source: "campus-locations",
			// 	minzoom: 8,
			// 	layout: {
			// 		'text-field': ['get', 'Campus'], 
			// 		'text-size': 19,
			// 		'text-anchor': 'top', 
			// 		'text-offset': [0, -1.7],
			// 		"text-font": [
			// 			"TradeGothic LT Bold"
			// 		],
			// 	},
			// 	paint: {
			// 		'text-color': defaultColor, 
			// 		'text-halo-color': haloColor, 
			// 		'text-halo-width': 2, 
			// 		'text-halo-blur': 1
			// 	},
			// });

			map.addLayer({
				id: "campus-points",
				type: "circle",
				source: "campus-locations",
				paint: {
					'circle-radius': [
						'sqrt', 
						[
							'/', 
								[
								'/', 
								['get', 'Programs'], 
								0.0015
							], 
							Math.PI 
						]
					],
					'circle-color': defaultColor,
					'circle-opacity': 0.8,
					"circle-stroke-color": haloColor,
					"circle-stroke-width": 2,
					'circle-stroke-opacity': 0.99
				}
			});

			map.addLayer({
				id: "campus-points-label",
				type: "symbol",
				source: "campus-locations",
				minzoom: 8,
				layout: {
					'text-field': ['get', 'Campus'], 
					'text-size': 19,
					'text-anchor': 'center', 
					'text-offset': [0, 0],
					"text-font": [
						"TradeGothic LT Bold"
					],
				},
				paint: {
					'text-color': defaultColor, 
					'text-halo-color': haloColor, 
					'text-halo-width': 2.5, 
					'text-halo-blur': 0.5,
					'text-opacity': 1
				},
			});

			
            map.on('zoom', () => {
                updateBeeswarm(map);
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


<div id="container">

	<div id="map">

		<div class="overlay">
			<a href="https://www.aspo.utoronto.ca" target="_blank" rel="noopener noreferrer">
				<img src={logo} alt="Overlay Image">
			</a>
		</div>

	</div>

	<div id="att">
		<p>ASPO gratefully acknowledges Jeff Allen and Scott McCallum at the <a href="https://schoolofcities.utoronto.ca/" target= "_blank" style="text-decoration: underline;">School of Cities</a> for designing this map. Data for the map are from <a href="https://openstreetmap.org" target= "_blank" style="text-decoration: underline;"> OpenStreetMap</a></p>
	</div>

</div>




<style>

	/* html, body {
		margin: 0;
		padding: 0;
		height: 100%;
	} */

	#container {
		display: flex;
		flex-direction: column;
		height: 100vh; 
	}

	#map {
		width: 100vw;
		height: 100vh;
		position: relative;
	}

	#att {
		flex-shrink: 0; 
		background-color: #6FC7EA; 
		padding: 4px;
		border-top: solid 1px var(--brandMedBlue);
		box-sizing: border-box; 
	}

	#att p {
		margin: 0;
		font-size: 11px;
		line-height: 1.3;
		color: var(--brandDarkBlue);
		text-align: right;
	}
	
	@media (max-width: 760px) {
		#att p {
			text-align: left; 
		}
	}

	a {
		color: var(--brandDarkBlue);
	}

	.overlay {
		position: absolute; 
		bottom: 0px; 
		right: 0; 
		width: 210px; 
		height: 30px; 
		background-color: #6fc7eab2;
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
		color: var(--brandDarkBlue);
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
