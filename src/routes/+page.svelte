<script>
	import { onMount } from "svelte";
	import Papa from "papaparse";
	import CampusMap from "$lib/CampusMap.svelte";
	import { CampusStore, TriCampusStore } from "$lib/stores";
	import "../assets/global.css";

	let campusData = [];
	let uniqueCampusData = [];
	let uniqueCampuses = [];
	let uniqueProgramTypes = [];
	let selectedProgramTypes = new Set();
	let filteredPrograms = [];
	let sortColumn = "Program Name";
	let sortOrder = "asc";

	let currentCampusStore = [];

	$: {
		currentCampusStore = $CampusStore;
		updateFilteredPrograms();
	}

	onMount( async() => {
		const csvFilePath = "/src/data/access_programs_inventory.csv";
		fetch(csvFilePath)
			.then((response) => response.text())
			.then((csvData) => {
				Papa.parse(csvData, {
					header: true,
					complete: (results) => {
						campusData = results.data;
						const campusMap = new Map();
						results.data.forEach((row) => {
							if (row.Campus && row.Longitude && row.Latitude) {
								const key = row.Campus.trim();
								if (!campusMap.has(key)) {
									campusMap.set(key, {
										longitude: parseFloat(row.Longitude),
										latitude: parseFloat(row.Latitude),
									});
								}
							}
						});
						uniqueCampusData = Array.from(
							campusMap,
							([campus, coords]) => ({
								campus,
								...coords,
							}),
						);
						uniqueCampuses = uniqueCampusData
							.map((data) => data.campus)
							.sort();
						uniqueProgramTypes = [
							...new Set(
								results.data
									.map((row) => row["Type of Program"])
									.filter(
										(type) =>
											type !==
											"Outreach & Engagement: Awareness_Career Exploration_Academic Supports; Access: Bridging_Transition_Pre-programs",
									),
							),
						].sort();
						updateFilteredPrograms();
					},
				});
			});
	});

	function handleCampusChange(event) {
		const campus = event.target.value;
		const currentStoreValues = $CampusStore;
		if (currentStoreValues.includes(campus)) {
			CampusStore.update((store) =>
				store.filter((item) => item !== campus),
			);
			if (campus === "Tri-campus") {
				TriCampusStore.set(false);
			}
		} else {
			CampusStore.update((store) => [...store, campus]);
			if (campus === "Tri-campus") {
				TriCampusStore.set(true);
			}
		}
	}

	function handleProgramTypeChange(event) {
		const programType = event.target.value;
		if (event.target.checked) {
			selectedProgramTypes.add(programType);
		} else {
			selectedProgramTypes.delete(programType);
		}
		updateFilteredPrograms();
	}

	function updateFilteredPrograms() {
		const currentCampuses = new Set(currentCampusStore);

		const allCampusesExceptTriCampus = uniqueCampuses.filter(
			(campus) => campus !== "Tri-campus",
		);
		const allSelectedExceptTriCampus = allCampusesExceptTriCampus.every(
			(campus) => currentCampuses.has(campus),
		);

		filteredPrograms = campusData
			.filter((row) => {
				const programType = row["Type of Program"];
				const includesSelectedType = Array.from(
					selectedProgramTypes,
				).some(
					(type) =>
						programType.includes(type) ||
						programType.includes(
							"Outreach & Engagement: Awareness_Career Exploration_Academic Supports; Access: Bridging_Transition_Pre-programs",
						),
				);

				const campusCondition =
					currentCampuses.size === 0 ||
					currentCampuses.has(row.Campus) ||
					(allSelectedExceptTriCampus && row.Campus === "Tri-campus");

				return (
					campusCondition &&
					(selectedProgramTypes.size === 0 || includesSelectedType)
				);
			})
			.sort((a, b) => {
				const compareA = a[sortColumn] || "";
				const compareB = b[sortColumn] || "";
				if (sortOrder === "asc") {
					return compareA.localeCompare(compareB);
				} else {
					return compareB.localeCompare(compareA);
				}
			});
	}

</script>

<div class="filters-container">
	
	<div class="program">
	<div class="filter">
		<label>Type of Program</label>
		{#each uniqueProgramTypes as programType}
			<div>
				<input
					type="checkbox"
					value={programType}
					on:change={handleProgramTypeChange}
					checked={selectedProgramTypes.has(programType)}
				/>
				{programType.replace(/_/g, " ")}
			</div>
		{/each}
	</div>
</div>

<div class="campus">
	<div class="filter">
		<label>Campus</label>
		<div class="campus-filters">
			{#each uniqueCampuses as campus}
				<div>
					<input
						type="checkbox"
						value={campus}
						on:change={handleCampusChange}
						checked={currentCampusStore.includes(campus)}
					/>
					{campus}
				</div>
			{/each}
		</div>
	</div>
</div>
</div>

<div class="container">

	{#if filteredPrograms.length > 0}
		<ul class="program-list">
			{#each filteredPrograms as program}
				<li>
					<strong>{program["Program Name"]}</strong><br />
					{program["Division"]}<br />
					{program["Campus"]}
				</li>
			{/each}
		</ul>
	{/if}

	<CampusMap />
</div>

<style>
	.container {
		display: flex;
		gap: 1rem; /* Space between the list and the map */
		margin: 1em 0 1em 1em;
	}

	.program-list {
		list-style-type: none;
		padding: 0;
		margin: 0;
		width: 40%;
		height: 80vh;
		overflow-y: auto;
		direction: rtl;
		text-align: left;
		border: 1px solid #ccc; /* Add a border around the program list */
	}

	.program-list li {
		margin: 1em;
		line-height: 1.5;
		border-bottom: 1px solid #ccc;
		padding-bottom: 1em;
		direction: ltr;
	}

	label {
		font-weight: bold;
		font-size: 1em;
		margin-bottom: .5em;
		display: block;
	}

	.filters-container {
		display: flex;
		gap: 1em;
		margin: 1em 0 0em 1em;
	}

	.program {
		width: 40%;
	}

	.campus {
		width: 60%;
	}

	.filter {
		flex: 1;
	}

	.campus-filters {
		display: grid;
		grid-template-columns: repeat(2, 0.2fr);
		grid-template-rows: repeat(2, auto);
		gap: 0rem;
	}
</style>
