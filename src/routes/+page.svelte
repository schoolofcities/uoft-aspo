<script>
	import { onMount } from "svelte";
	import Papa from "papaparse";
    import CampusMap from "$lib/CampusMap.svelte";

	let campusData = [];
	let uniqueCampusData = [];
	let uniqueCampuses = [];
	let uniqueProgramTypes = [];
	let uniqueDivisions = [];
	let selectedCampuses = new Set();
	let selectedProgramTypes = new Set();
	let selectedDivision = "";
	let filteredPrograms = [];
	let sortColumn = "Program Name";
	let sortOrder = "asc";

	onMount(async() => {
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

						// Log the unique campus names to the console
						console.log("Unique Campuses:", uniqueCampuses);

						uniqueProgramTypes = [
							...new Set(
								results.data.map(
									(row) => row["Type of Program"],
								),
							),
						].sort();
						uniqueDivisions = [
							...new Set(results.data.map((row) => row.Division)),
						].sort();

						updateFilteredPrograms();
					},
				});
			});
	});

	function handleCampusChange(event) {
		const campus = event.target.value;
		if (event.target.checked) {
			selectedCampuses.add(campus);
		} else {
			selectedCampuses.delete(campus);
		}
		updateFilteredPrograms();
	}

	function handleDivisionChange(event) {
		selectedDivision = event.target.value;
		updateFilteredPrograms();
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
		filteredPrograms = campusData
			.filter(
				(row) =>
					(selectedCampuses.size === 0 ||
						selectedCampuses.has(row.Campus)) &&
					(selectedProgramTypes.size === 0 ||
						selectedProgramTypes.has(row["Type of Program"])) &&
					(selectedDivision === "" ||
						row.Division === selectedDivision),
			)
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
			<input type="checkbox" value={programType} on:change={handleProgramTypeChange} />
			{programType}
		</div>
	{/each}
</div>

<div class="filters-container">
	<div class="filter">
		<label>Campus</label>
		{#each uniqueCampuses as campus}
		<div>
			<input type="checkbox" value={campus} on:change={handleCampusChange} />
			{campus}
		</div>
		{/each}
	</div>

	<div class="filter">
		<label for="division">Division</label>
		<select id="division" bind:value={selectedDivision} on:change={handleDivisionChange}>
			<option value="">Select a division</option>
			{#each uniqueDivisions as division}
				<option value={division}>{division}</option>
			{/each}
		</select>
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
			{#each filteredPrograms as { 'Program Name': programName, Campus, Division }}
				<tr>
					<td>{programName}</td>
					<td>{Campus}</td>
					<td>{Division}</td>
				</tr>
			{/each}
		</tbody>
	</table>
{:else}
	<p>No programs matching selected criteria.</p>
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
