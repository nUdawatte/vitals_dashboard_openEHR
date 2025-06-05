<script lang="ts">
	import { tick } from 'svelte';
	import { Chart } from 'chart.js/auto';
	import { goto } from '$app/navigation';
	import localwebtemplate from "../vital_signs_nuwan_udawatte.v2.json";
	import "../../styles/vitals.css";
	import type { ChartDataset } from 'chart.js';

	const baseURL = "https://openehr-bootcamp.medblocks.com/ehrbase/rest/openehr/v1/query/aql";

	let ehrId = "6adc37f8-fff8-4fe4-a988-327b9fe6d5ac";
	let data: any[] = [];
	let activeView: 'my-template' | 'all-readings' = 'my-template';
	let selectedVital: 'blood_pressure' | 'spo2' | 'heart_rate' | 'body_weight' | 'height' = 'blood_pressure';
	let chartCanvas: HTMLCanvasElement | null = null;
	let chart: Chart;
	let weightUnit: 'kg' | 'g' | 'lb' = 'kg';

	// Title reactive
	$: chartTitle =
		selectedVital === 'blood_pressure' ? 'Blood Pressure' :
		selectedVital === 'spo2' ? 'SpO₂' :
		selectedVital === 'heart_rate' ? 'Heart Rate' :
		selectedVital === 'body_weight' ? 'Body Weight' :
		selectedVital === 'height' ? 'Height' : 'Vital';

	// Trigger refresh reactively
	$: if (selectedVital || activeView) {
		setTimeout(() => {
			loadVitals();
		}, 0);
	}

	function convertWeight(value: number, fromUnit: string, toUnit: string): number {
		if (fromUnit === toUnit) return value;
		let valueInKg = value;
		switch (fromUnit) {
			case 'g': valueInKg = value / 1000; break;
			case 'lb': valueInKg = value * 0.453592; break;
		}
		switch (toUnit) {
			case 'g': return valueInKg * 1000;
			case 'lb': return valueInKg / 0.453592;
			default: return valueInKg;
		}
	}

	const queries = {
		blood_pressure: `
			SELECT 
				bp/data[at0001]/events[at0006]/data[at0003]/items[at0004]/value/magnitude as systolic,
				bp/data[at0001]/events[at0006]/data[at0003]/items[at0005]/value/magnitude as diastolic,
				c/context/start_time/value as timestamp,
				c/archetype_details/template_id/value as template_id  
			FROM EHR e CONTAINS COMPOSITION c 
			CONTAINS OBSERVATION bp [openEHR-EHR-OBSERVATION.blood_pressure.v2] 
			WHERE {TEMPLATE_FILTER}
			ORDER BY c/context/start_time/value
		`,
		spo2: `
			SELECT 
				o/data[at0001]/events[at0002]/data[at0003]/items[at0006]/value/numerator as spo2,
				c/context/start_time/value as timestamp,
				c/archetype_details/template_id/value as template_id
			FROM EHR e CONTAINS COMPOSITION c 
			CONTAINS OBSERVATION o [openEHR-EHR-OBSERVATION.pulse_oximetry.v1]
			WHERE {TEMPLATE_FILTER}
			ORDER BY c/context/start_time/value
		`,
		heart_rate: `
			SELECT 
				o/data[at0002]/events[at0003]/data[at0001]/items[at0004.1]/value/magnitude as heart_rate,
				c/context/start_time/value as timestamp,
				c/archetype_details/template_id/value as template_id
			FROM EHR e CONTAINS COMPOSITION c 
			CONTAINS OBSERVATION o [openEHR-EHR-OBSERVATION.heartbeat-pulse.v0]
			WHERE {TEMPLATE_FILTER}
			ORDER BY c/context/start_time/value
		`,
		body_weight: `
			SELECT 
				o/data[at0002]/events[at0003]/data[at0001]/items[at0004]/value/magnitude as weight,
				o/data[at0002]/events[at0003]/data[at0001]/items[at0004]/value/units as unit,
				c/context/start_time/value as timestamp,
				c/archetype_details/template_id/value as template_id
			FROM EHR e CONTAINS COMPOSITION c 
			CONTAINS OBSERVATION o [openEHR-EHR-OBSERVATION.body_weight.v2]
			WHERE {TEMPLATE_FILTER}
			ORDER BY c/context/start_time/value
		`,
		height: `
			SELECT 
				o/data[at0001]/events[at0002]/data[at0003]/items[at0004]/value/magnitude as height,
				c/context/start_time/value as timestamp,
				c/archetype_details/template_id/value as template_id
			FROM EHR e CONTAINS COMPOSITION c 
			CONTAINS OBSERVATION o [openEHR-EHR-OBSERVATION.height.v2]
			WHERE {TEMPLATE_FILTER}
			ORDER BY c/context/start_time/value
		`
	};

	async function loadVitals() {
		const templateFilter =
			activeView === 'my-template'
				? `c/archetype_details/template_id/value = '${localwebtemplate.templateId}' AND e/ehr_id/value = '${ehrId}'`
				: `e/ehr_id/value = '${ehrId}'`;

		const query = queries[selectedVital].replace('{TEMPLATE_FILTER}', templateFilter);
		const response = await fetch(baseURL, {
			method: 'POST',
			headers: { 'Content-Type': 'application/json' },
			body: JSON.stringify({ q: query })
		});

		const responseData = await response.json();
		data = responseData.rows.map((row: any) => {
			if (selectedVital === 'blood_pressure') {
				return { systolic: row[0], diastolic: row[1], timestamp: row[2], template_id: row[3] };
			} else if (selectedVital === 'spo2') {
				return { spo2: row[0], timestamp: row[1], template_id: row[2] };
			} else if (selectedVital === 'heart_rate') {
				return { heart_rate: row[0], timestamp: row[1], template_id: row[2] };
			} else if (selectedVital === 'body_weight') {
				return { weight: row[0], unit: row[1], timestamp: row[2], template_id: row[3] };
			} else {
				return { height: row[0], timestamp: row[1], template_id: row[2] };
			}
		});

		await tick();
		updateChart();
	}

	function getAverage(metric: 'systolic' | 'diastolic' | 'spo2' | 'heart_rate' | 'weight' | 'height') {
		const values =
			metric === 'weight'
				? data.map(d => convertWeight(d.weight, d.unit, weightUnit)).filter(v => !isNaN(v))
				: data.map(d => d[metric]).filter(v => typeof v === 'number' && !isNaN(v));
		return values.length ? (values.reduce((a, b) => a + b, 0) / values.length).toFixed(1) : 'N/A';
	}

	async function updateChart() {
		await tick();
		if (!chartCanvas) return;
		if (chart) chart.destroy();

		const labels = data.map(row => new Date(row.timestamp).toLocaleString());
		let datasets: ChartDataset<'line'>[] = [];

		if (selectedVital === 'blood_pressure') {
			datasets = [
				{
					label: 'Systolic (mmHg)',
					data: data.map(row => row.systolic),
					borderColor: '#EF4444',
					backgroundColor: 'rgba(239, 68, 68, 0.1)',
					tension: 0.3,
					pointRadius: 4,
					fill: true
				},
				{
					label: 'Diastolic (mmHg)',
					data: data.map(row => row.diastolic),
					borderColor: '#3B82F6',
					backgroundColor: 'rgba(59, 130, 246, 0.1)',
					tension: 0.3,
					pointRadius: 4,
					fill: true
				}
			];
		} else if (selectedVital === 'spo2') {
			datasets = [
				{
					label: 'SpO₂ (%)',
					data: data.map(row => row.spo2),
					borderColor: '#10B981',
					backgroundColor: 'rgba(16, 185, 129, 0.1)',
					tension: 0.3,
					pointRadius: 4,
					fill: true
				}
			];
		} else if (selectedVital === 'heart_rate') {
			datasets = [
				{
					label: 'Heart Rate (bpm)',
					data: data.map(row => row.heart_rate),
					borderColor: '#F59E0B',
					backgroundColor: 'rgba(245, 158, 11, 0.1)',
					tension: 0.3,
					pointRadius: 4,
					fill: true
				}
			];
		} else if (selectedVital === 'body_weight') {
			datasets = [
				{
					label: `Weight (${weightUnit})`,
					data: data.map(row => convertWeight(row.weight, row.unit, weightUnit)),
					borderColor: '#8B5CF6',
					backgroundColor: 'rgba(139, 92, 246, 0.1)',
					tension: 0.3,
					pointRadius: 4,
					fill: true
				}
			];
		} else if (selectedVital === 'height') {
			datasets = [
				{
					label: `Height (cm)`,
					data: data.map(row => row.height),
					borderColor: '#06B6D4',
					backgroundColor: 'rgba(6, 182, 212, 0.1)',
					tension: 0.3,
					pointRadius: 4,
					fill: true
				}
			];
		}

		chart = new Chart(chartCanvas, {
			type: 'line',
			data: { labels, datasets },
			options: {
				responsive: true,
				maintainAspectRatio: false,
				scales: {
					y: { beginAtZero: false, grid: { color: '#e5e7eb' } },
					x: { grid: { color: '#f3f4f6' } }
				},
				plugins: {
					tooltip: {
						mode: 'index',
						intersect: false,
						callbacks: {
							afterBody: (ctx) => `Template: ${data[ctx[0]?.dataIndex]?.template_id ?? ''}`
						}
					},
					legend: {
						labels: {
							color: '#374151',
							boxWidth: 12
						}
					}
				}
			}
		});
	}
</script>



<!-- HTML UI BLOCK -->
<div class="p-6 bg-gradient-to-br from-sky-50 to-white min-h-screen space-y-8">
    <!-- Header -->
    <div class="flex flex-col md:flex-row justify-between items-center gap-4">
      <h1 class="text-3xl font-bold text-sky-700">📈 Vitals Overview</h1>
      <button class="bg-blue-600 text-white px-5 py-2 rounded-lg hover:bg-blue-700 shadow" on:click={() => goto('/')}>
        <i class="fas fa-clipboard-list"></i> Back to Form
      </button>
    </div>
  
    <!-- Controls -->
    <div class="flex flex-wrap items-center gap-4">
        <div class="flex items-center gap-3 relative group">
            <label
            for="vital-select"
            class="text-base font-semibold text-sky-700 uppercase tracking-wide"
          >
          📈  Select Vital Type
          </label>
            <div class="relative">
                <select
                id="vital-select"
                bind:value={selectedVital}
                class="peer block appearance-none w-48 bg-white border border-gray-300 text-gray-800 py-2 px-4 pr-10 rounded-lg leading-tight focus:outline-none focus:ring-2 focus:ring-blue-400 focus:border-blue-400 shadow transition duration-150"
                >
                <option value="blood_pressure"> 🩺 Blood Pressure</option>
                <option value="spo2">🩸 SpO₂</option>
                <option value="heart_rate">❤️ Heart Rate</option>
                <option value="body_weight">⚖️ Body Weight</option>
                <option value="height">📏 Height</option>
                </select>
                <div class="pointer-events-none absolute inset-y-0 right-0 flex items-center px-2 text-gray-500">
                <svg class="w-4 h-4 fill-current" viewBox="0 0 20 20">
                    <path d="M7 7l3-3 3 3m0 6l-3 3-3-3" />
                </svg>
                </div>
            </div>
            </div>
          
  
      {#if selectedVital === 'body_weight'}
        <div class="flex items-center gap-3">
          <label for="unit-select" class="font-medium text-gray-700">⚖️ Unit:</label>
          <select
            id="unit-select"
            class="block appearance-none w-full bg-white border border-gray-300 text-gray-700 py-2 px-4 pr-8 rounded leading-tight focus:outline-none focus:ring-2 focus:ring-blue-300 shadow-sm"
            bind:value={weightUnit}
          >
            <option value="kg">Kilograms (kg)</option>
            <option value="g">Grams (g)</option>
            <option value="lb">Pounds (lb)</option>
          </select>
        </div>
      {/if}
  
      <div class="flex gap-3">
        <button class={`px-4 py-2 rounded-full text-sm font-medium ${activeView === 'my-template' ? 'bg-blue-700 text-white' : 'bg-white border border-blue-300 text-blue-700'}`} on:click={() => (activeView = 'my-template')}>
          🧾 My Template
        </button>
        <button class={`px-4 py-2 rounded-full text-sm font-medium ${activeView === 'all-readings' ? 'bg-blue-700 text-white' : 'bg-white border border-blue-300 text-blue-700'}`} on:click={() => (activeView = 'all-readings')}>
          🌐 All Readings
        </button>
      </div>
    </div>
  
    {#if data.length > 0}
      <!-- Summary Cards -->
      <div class="flex gap-4 flex-wrap text-sm">
        {#if selectedVital === 'blood_pressure'}
          <div class="bg-white shadow px-4 py-3 rounded-xl border-l-4 border-red-500">
            <p class="text-gray-600 font-medium">Avg Systolic</p>
            <p class="text-lg font-semibold text-red-600">{getAverage('systolic')} mmHg</p>
          </div>
          <div class="bg-white shadow px-4 py-3 rounded-xl border-l-4 border-blue-500">
            <p class="text-gray-600 font-medium">Avg Diastolic</p>
            <p class="text-lg font-semibold text-blue-600">{getAverage('diastolic')} mmHg</p>
          </div>
        {:else if selectedVital === 'spo2'}
          <div class="bg-white shadow px-4 py-3 rounded-xl border-l-4 border-emerald-500">
            <p class="text-gray-600 font-medium">Avg SpO₂</p>
            <p class="text-lg font-semibold text-emerald-600">{getAverage('spo2')}%</p>
          </div>
        {:else if selectedVital === 'heart_rate'}
          <div class="bg-white shadow px-4 py-3 rounded-xl border-l-4 border-yellow-500">
            <p class="text-gray-600 font-medium">Avg Heart Rate</p>
            <p class="text-lg font-semibold text-yellow-600">{getAverage('heart_rate')} bpm</p>
          </div>
        {:else if selectedVital === 'body_weight'}
          <div class="bg-white shadow px-4 py-3 rounded-xl border-l-4 border-purple-500">
            <p class="text-gray-600 font-medium">Avg Weight</p>
            <p class="text-lg font-semibold text-purple-600">{getAverage('weight')} {weightUnit}</p>
          </div>
        {:else if selectedVital === 'height'}
          <div class="bg-white shadow px-4 py-3 rounded-xl border-l-4 border-cyan-500">
            <p class="text-gray-600 font-medium">Avg Height</p>
            <p class="text-lg font-semibold text-cyan-600">{getAverage('height')} cm</p>
          </div>
        {/if}
        <div class="bg-white shadow px-4 py-3 rounded-xl border-l-4 border-gray-400">
          <p class="text-gray-600 font-medium">Readings</p>
          <p class="text-lg font-semibold text-gray-800">{data.length}</p>
        </div>
      </div>
  
      <!-- Chart + Table -->
      <div class="grid grid-cols-1 xl:grid-cols-2 gap-6 mt-6">
        <div class="bg-white rounded-2xl p-6 shadow-xl h-[500px]">
          <h2 class="text-lg font-semibold text-gray-800 mb-4">
            📊 {chartTitle} Trend
          </h2>
          <canvas bind:this={chartCanvas} class="w-full h-full"></canvas>
        </div>
  
        <div class="bg-white rounded-2xl p-6 shadow-xl overflow-auto">
          <h2 class="text-lg font-semibold text-gray-800 mb-4">📋 Detailed Records</h2>
          <table class="min-w-full text-sm border-separate border-spacing-y-2">
            <thead class="bg-blue-100 text-blue-800">
              <tr>
                <th class="p-3 text-left">🧾 Template</th>
                <th class="p-3 text-left">⏰ Time</th>
                {#if selectedVital === 'blood_pressure'}
                  <th class="p-3 text-left">🔺 Systolic</th>
                  <th class="p-3 text-left">🔻 Diastolic</th>
                {:else if selectedVital === 'spo2'}
                  <th class="p-3 text-left">🫁 SpO₂ (%)</th>
                {:else if selectedVital === 'heart_rate'}
                  <th class="p-3 text-left">❤️ Heart Rate (bpm)</th>
                {:else if selectedVital === 'body_weight'}
                  <th class="p-3 text-left">⚖️ Weight ({weightUnit})</th>
                {:else if selectedVital === 'height'}
                  <th class="p-3 text-left">📏 Height (cm)</th>
                {/if}
              </tr>
            </thead>
            <tbody>
              {#each data as row}
                <tr class="bg-gray-50 hover:bg-blue-50 rounded-lg">
                  <td class="p-3">{row.template_id}</td>
                  <td class="p-3">{new Date(row.timestamp).toLocaleString()}</td>
                  {#if selectedVital === 'blood_pressure'}
                    <td class="p-3 font-semibold text-red-600">{row.systolic}</td>
                    <td class="p-3 font-semibold text-blue-600">{row.diastolic}</td>
                  {:else if selectedVital === 'spo2'}
                    <td class="p-3 font-semibold text-emerald-600">{row.spo2}</td>
                  {:else if selectedVital === 'heart_rate'}
                    <td class="p-3 font-semibold text-yellow-600">{row.heart_rate}</td>
                  {:else if selectedVital === 'body_weight'}
                    <td class="p-3 font-semibold text-purple-600">{convertWeight(row.weight, row.unit, weightUnit).toFixed(1)}</td>
                  {:else if selectedVital === 'height'}
                    <td class="p-3 font-semibold text-cyan-600">{row.height}</td>
                  {/if}
                </tr>
              {/each}
            </tbody>
          </table>
        </div>
      </div>
    {:else}
      <p class="text-gray-600 mt-6">No vitals data found for selected filter.</p>
    {/if}
  </div>
  
