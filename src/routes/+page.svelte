<script lang="ts">
	export const ssr = false;
	import "medblocks-ui";
	import "medblocks-ui/dist/styles.js";
	import "../styles/vitals.css";
	import localwebtemplate from "./vital_signs_nuwan_udawatte.v2.json";
	import { onMount } from "svelte";
	import { Toaster, toast } from "svelte-sonner";
    import { goto } from "$app/navigation";

	let ehrId = "6adc37f8-fff8-4fe4-a988-327b9fe6d5ac";
	let openehrBaseUrl = "https://openehr-bootcamp.medblocks.com/ehrbase/rest/openehr/v1";

	let submissionForm: any;
	let vitals: any[] = [];
	let formKey = 0;
	let updatingVitalUid: string | null = null;

	let filterStart = '';
	let filterEnd = '';
	let minSystolic: number | null = null;
	let maxSystolic: number | null = null;

	function resetFilters() {
		filterStart = '';
		filterEnd = '';
		minSystolic = null;
		maxSystolic = null;
	}

	$: filteredVitals = vitals.filter(v => {
		const time = new Date(v.start_time);
		const systolic = v.systolic ?? 0;
		const afterStart = !filterStart || time >= new Date(filterStart);
		const beforeEnd = !filterEnd || time <= new Date(filterEnd);
		const aboveMin = minSystolic === null || systolic >= minSystolic;
		const belowMax = maxSystolic === null || systolic <= maxSystolic;
		return afterStart && beforeEnd && aboveMin && belowMax;
	});

	async function submitVitalData() {
		try {
			const composition = submissionForm.export();
			if (!composition) {
				toast.error("Form is empty or invalid.");
				return;
			}

			if (updatingVitalUid) {
				const compositionId = updatingVitalUid.split("::")[0];
				await toast.promise(
					fetch(`${openehrBaseUrl}/ehr/${ehrId}/composition/${compositionId}?templateId=${localwebtemplate.templateId}`, {
						method: "PUT",
						headers: {
							"Content-Type": "application/openehr.wt.flat.schema+json",
							Accept: "application/openehr.wt.flat.schema+json",
							Prefer: "return=representation",
							"If-Match": updatingVitalUid
						},
						body: JSON.stringify(composition)
					}).then(async (res) => {
						const responseData = await res.json();
						if (!res.ok) throw new Error(responseData.message || "Update failed");
						vitals = await getVitals() || [];
						formKey += 1;
						updatingVitalUid = null;
					}),
					{ loading: "Updating vitals...", success: "Vitals updated successfully!", error: err => err instanceof Error ? err.message : "Update failed" }
				);
			} else {
				await toast.promise(
					fetch(`${openehrBaseUrl}/ehr/${ehrId}/composition?templateId=${localwebtemplate.templateId}`, {
						method: "POST",
						headers: {
							"Content-Type": "application/openehr.wt.flat.schema+json",
							Accept: "application/openehr.wt.flat.schema+json",
							Prefer: "return=representation"
						},
						body: JSON.stringify(composition)
					}).then(async (res) => {
						const responseData = await res.json();
						if (!res.ok) throw new Error(responseData.message || "Submission failed");
						vitals = await getVitals() || [];
						formKey += 1;
					}),
					{ loading: "Submitting form...", success: "Form submitted successfully!", error: err => err instanceof Error ? err.message : "Submission failed" }
				);
			}
		} catch (err) {
			console.error("Unexpected error:", err);
		}
	}

	async function getVitals() {
		try {
            const response = await fetch(`${openehrBaseUrl}/query/aql`, {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
                q: `SELECT 
                    c/uid/value, 
                    c/context/start_time/value,
                    o/data[at0002]/events[at0003]/data[at0001]/items[at0004.1]/value/magnitude,
                    o2/data[at0001]/events[at0006]/data[at0003]/items[at0004]/value/magnitude,
                    o2/data[at0001]/events[at0006]/data[at0003]/items[at0005]/value/magnitude,
                    o3/data[at0001]/events[at0002]/data[at0003]/items[at0006]/value/numerator,
                    o4/data[at0002]/events[at0003]/data[at0001]/items[at0004]/value/magnitude
                FROM EHR e CONTAINS COMPOSITION c CONTAINS (
                    OBSERVATION o[openEHR-EHR-OBSERVATION.heartbeat-pulse.v0] AND
                    OBSERVATION o2[openEHR-EHR-OBSERVATION.blood_pressure.v2] AND
                    OBSERVATION o3[openEHR-EHR-OBSERVATION.pulse_oximetry.v1] AND
                    OBSERVATION o4[openEHR-EHR-OBSERVATION.body_weight.v2]
                )
                WHERE c/archetype_details/template_id/value = '${localwebtemplate.templateId}' 
                    AND e/ehr_id/value = '${ehrId}'`
            })
            });
 
			if (response.ok) {
				const data = await response.json();
				return data.rows.map((row: any) => ({
					uid: row[0], start_time: row[1], pulse: row[2], systolic: row[3], diastolic: row[4], spo2: row[5]
				}));
			}
		} catch (err) {
			console.error("Fetch error:", err);
		}
	}

	async function deleteVitalData(uid: string) {
		await toast.promise(
			fetch(`${openehrBaseUrl}/ehr/${ehrId}/composition/${uid}`, {
				method: "DELETE",
				headers: { "Content-Type": "application/json" }
			}).then(async (res) => {
				if (!res.ok) throw new Error("Failed to delete");
				vitals = (await getVitals()) || [];
			}),
			{ loading: "Deleting entry...", success: "Entry deleted successfully!", error: err => err instanceof Error ? err.message : "Something went wrong" }
		);
	}

	async function bindVitals(uid: string) {
		const toastId = toast.loading("Loading data to the form...");
		try {
			const response = await fetch(`${openehrBaseUrl}/ehr/${ehrId}/composition/${uid}`, {
				method: "GET",
				headers: { Accept: "application/openehr.wt.flat.schema+json" }
			});
			if (response.ok) {
				submissionForm.import(await response.json());
				updatingVitalUid = uid;
				submissionForm.scrollIntoView({ behavior: "smooth" });
			} else throw new Error("Failed to load data");
		} catch (err) {
			console.error(err);
		} finally {
			toast.dismiss(toastId);
		}
	}

	function cancelEdit() {
		formKey += 1;
		updatingVitalUid = null;
		toast.info("Edit cancelled");
	}

	onMount(async () => {
		vitals = (await getVitals()) || [];
	});
</script>

<Toaster />

<div class="flex justify-between items-center p-4 border-b bg-white shadow-sm">
	<h1 class="text-xl font-semibold text-gray-800">Vital Signs Management</h1>
	<button class="text-base bg-blue-600 text-white px-6 py-3 rounded-lg hover:bg-blue-700 transition font-medium shadow" on:click={() => goto('/dashboard')}>
		📊 Go to Dashboard
	</button>
</div>

<div class="flex flex-col xl:flex-row min-h-[90vh] bg-gradient-to-br from-sky-50 to-white">
	<!-- Form Section -->
	<div class="xl:w-1/2 p-8 bg-white border-r border-gray-200 shadow-inner overflow-y-auto">
		<h1 class="text-2xl font-bold mb-2 text-blue-700 flex items-center gap-2">
			<i class="fas fa-heart-pulse"></i>
			{updatingVitalUid ? "Update Vitals" : "New Vitals Entry"}
		</h1>
		<p class="text-sm text-gray-600 mb-4">
			{updatingVitalUid ? "Edit the selected vitals entry and press Update to save changes." : "Fill out the form below to record new vital signs."}
		</p>
		<div class="bg-blue-50 border border-blue-100 rounded-xl p-6 shadow-sm">
			{#key formKey}
				<mb-auto-form bind:this={submissionForm} webtemplate={JSON.stringify(localwebtemplate)}></mb-auto-form>
			{/key}
			<div class="flex flex-wrap gap-4 mt-6">
				<button class="px-4 py-2 bg-blue-600 text-white rounded-lg shadow hover:bg-blue-700 transition" on:click={submitVitalData}>
					<i class="fas fa-paper-plane mr-2"></i>{updatingVitalUid ? "Update" : "Submit"}
				</button>
				{#if updatingVitalUid}
					<button class="px-4 py-2 text-gray-500 hover:text-orange-500 transition" on:click={cancelEdit}>
						<i class="fas fa-xmark mr-1"></i>Cancel Edit
					</button>
				{/if}
			</div>
		</div>
	</div>

	<!-- Entries Section -->
	<div class="xl:w-1/2 p-8 bg-gradient-to-br from-sky-50 to-white overflow-y-auto relative">
		<h2 class="text-xl font-bold text-blue-700 mb-4 flex items-center gap-2">
			<i class="fas fa-notes-medical"></i> Previous Entries
		</h2>
		<div class="bg-white p-4 rounded-xl shadow mb-6 space-y-3">
			<div class="grid grid-cols-2 md:grid-cols-4 gap-3 text-sm">
				<!-- svelte-ignore a11y_label_has_associated_control -->
				<div><label class="block text-gray-600">From</label><input type="date" class="w-full border border-gray-300 rounded px-2 py-1" bind:value={filterStart} /></div>
				<!-- svelte-ignore a11y_label_has_associated_control -->
				<div><label class="block text-gray-600">To</label><input type="date" class="w-full border border-gray-300 rounded px-2 py-1" bind:value={filterEnd} /></div>
				<!-- svelte-ignore a11y_label_has_associated_control -->
				<div><label class="block text-gray-600">Min Systolic</label><input type="number" class="w-full border border-gray-300 rounded px-2 py-1" bind:value={minSystolic} /></div>
				<!-- svelte-ignore a11y_label_has_associated_control -->
				<div><label class="block text-gray-600">Max Systolic</label><input type="number" class="w-full border border-gray-300 rounded px-2 py-1" bind:value={maxSystolic} /></div>
			</div>
			<button class="text-sm text-blue-600 hover:underline mt-2" on:click={resetFilters}>Reset Filters</button>
		</div>
		{#if filteredVitals.length > 0}
			<div class="grid gap-4">
				{#each filteredVitals as vital, index (vital.uid)}
					<div class="bg-white rounded-xl shadow-md p-4 border border-gray-100 animate-fadeInUp">
						<div class="flex justify-between items-center">
							<div class="text-sm font-semibold text-gray-800 flex items-center gap-2">
								<i class="fas fa-stethoscope text-blue-500"></i> Entry {index + 1}
							</div>
							<div class="text-xs text-gray-400">🆔 {vital.uid}</div>
						</div>
						<div class="mt-2 text-sm text-gray-700 space-y-1">
							<div>📅 <strong>{new Date(vital.start_time).toLocaleString()}</strong></div>
							<div>💓 Pulse: {vital.pulse ?? "N/A"} /min</div>
							<div>🩺 BP: {vital.systolic ?? "?"}/{vital.diastolic ?? "?"} mmHg</div>
							<div>🩸 SpO₂: {vital.spo2 ?? "N/A"}%</div>
						</div>
						<div class="mt-3 flex gap-4">
							<button class="text-sm text-blue-600 hover:underline" on:click={() => bindVitals(vital.uid)}>✏️ Edit</button>
							<button class="text-sm text-red-600 hover:underline" on:click={() => deleteVitalData(vital.uid)}>🗑️ Delete</button>
						</div>
					</div>
				{/each}
			</div>
		{:else}
			<p class="text-gray-500">📭 No entries match the filter.</p>
		{/if}
		<!-- <button class="fixed bottom-6 right-6 bg-blue-600 hover:bg-blue-700 text-white px-5 py-3 rounded-full shadow-xl flex items-center gap-2" on:click={cancelEdit} title="New Vitals Entry">
			<i class="fas fa-plus"></i> New Entry
		</button> -->
	</div>
</div>

<style>
@keyframes fadeInUp {
	from { opacity: 0; transform: translateY(10px); }
	to { opacity: 1; transform: translateY(0); }
}
.animate-fadeInUp {
	animation: fadeInUp 0.4s ease-out;
}
</style>
