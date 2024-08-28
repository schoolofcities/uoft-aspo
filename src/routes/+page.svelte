<script>
	import { onMount } from "svelte";
	import Papa from "papaparse";
	import CampusMap from "$lib/CampusMap.svelte";
	import { CampusStore } from "$lib/stores";
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

	onMount(() => {
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
		} else {
			CampusStore.update((store) => [...store, campus]);
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
				selectedProgramTypes
			).some(
				(type) =>
					programType.includes(type) ||
					programType.includes(
						"Outreach & Engagement: Awareness_Career Exploration_Academic Supports; Access: Bridging_Transition_Pre-programs"
					)
			);

			const campusCondition =
				(currentCampuses.size === 0 ||
					currentCampuses.has(row.Campus)) ||
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

	function sortBy(column) {
		if (sortColumn === column) {
			sortOrder = sortOrder === "asc" ? "desc" : "asc";
		} else {
			sortColumn = column;
			sortOrder = "asc";
		}
		updateFilteredPrograms();
	}
</script>

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
			{programType}
		</div>
	{/each}
</div>
<div class="filters-container">
	<div class="filter">
		<label>Campus</label>
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

<CampusMap />

{#if filteredPrograms.length > 0}
	<table>
		<thead>
			<tr>
				<th on:click={() => sortBy("Program Name")}>
					Program Name {sortColumn === "Program Name"
						? sortOrder === "asc"
							? "▲"
							: "▼"
						: ""}
				</th>
				<th on:click={() => sortBy("Campus")}>
					Campus {sortColumn === "Campus"
						? sortOrder === "asc"
							? "▲"
							: "▼"
						: ""}
				</th>
				<th on:click={() => sortBy("Division")}>
					Division {sortColumn === "Division"
						? sortOrder === "asc"
							? "▲"
							: "▼"
						: ""}
				</th>
			</tr>
		</thead>
		<tbody>
			{#each filteredPrograms as program}
				<tr>
					<td>{program["Program Name"]}</td>
					<td>{program.Campus}</td>
					<td>{program.Division}</td>
				</tr>
			{/each}
		</tbody>
	</table>
{/if}

<style>
	.filters-container {
		display: flex;
		gap: 1rem;
		margin-bottom: 1rem;
	}

	.filter {
		flex: 1;
	}

	table {
		width: 100%;
		border-collapse: collapse;
		margin-top: 1rem;
		cursor: pointer;
	}

	th,
	td {
		border: 1px solid #ddd;
		padding: 8px;
	}

	th {
		background-color: #f2f2f2;
	}

	th {
		text-align: left;
	}
</style>
