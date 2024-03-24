<script>
	import { base } from "$app/paths";
	import Timezone from "$lib/components/timezone.svelte";
	import { csvParse, csvParseRows } from "d3";
	import { onMount } from "svelte";
	import { derived, readable, writable } from "svelte/store";


	/**
	 * zone_name,country_code,abbreviation,time_start,gmt_offset,dst
	 * @typedef {Object} Timezone
	 * @property {string} zone_name
	 * @property {string} country_code
	 * @property {string} abbreviation
	 * @property {EpochTimeStamp} time_start
	 * @property {number} gmt_offset in milliseconds
	 * @property {boolean} dst Whether daylight savings is in effect
	 */

	/**
	 * Load the list of timezones from time_zone.csv
	 * @returns {Promise<Record<string, Timezone[]>>}
	 */
	async function loadTimezones() {
		const response = await fetch(`${base}/time_zone.csv`);
		const text = await response.text();
		const timezoneEntries = csvParseRows(text);

		/** @type {Record<string, Timezone[]>} */
		let _timezones = {};
		for (let entry of timezoneEntries) {
			let [zone_name, country_code, abbreviation, time_start, gmt_offset, dst] = entry;
			if (!_timezones[zone_name]) {
				_timezones[zone_name] = [];
			}
			_timezones[zone_name].push({
				zone_name,
				country_code,
				abbreviation,
				time_start: +time_start * 1000,
				gmt_offset: +gmt_offset * 1000,
				dst: dst === "1"
			});
		}

		// Sort timezones by time_start from latest to earliest
		for (let zone in _timezones) {
			_timezones[zone].sort((a, b) => b.time_start - a.time_start);
		}

		timezones = readable(_timezones);

		return _timezones;
	}



	async function loadCountries() {
		const response = await fetch(`${base}/country.csv`);
		const text = await response.text();
		const countryEntries = csvParseRows(text);

		/** @type {Record<string, string>} */
		let _countries = {};
		for (let entry of countryEntries) {
			let [country_code, country_name] = entry;
			_countries[country_code] = country_name;
		}
		
		countries = readable(_countries);

		return _countries;

	}

	let countries = readable({});


	async function loadCities() {
		const response = await fetch(`${base}/worldcities.csv`);
		const text = await response.text();
		const cityEntries = csvParse(text);

		cities = readable(cityEntries);

		return cityEntries;
	
	}

	let cities = readable([]);


	/**
	 * A readable store containing the list of timezones.
	 * @type {import("svelte/store").Readable<Record<string, Timezone[]>>}
	 */
	let timezones = readable({});

	onMount(()=>{
		loadTimezones();
		loadCountries();
		loadCities();
	});

	

	/**
	 * A derived store containing searchable timezone names and their respective timezones.
	 * @type {import("svelte/store").Readable<Record<string, string>>}
	 */
	$: timezoneNames = derived([timezones, countries], ([$timezones, $countries]) => {

		let countryToTimezoneMap = Object.entries($countries).map(
			([country_code, country_name])=>
			[country_name, Object.values($timezones).map(tzs=>tzs[0]).find(tz=>tz.country_code == country_code)?.zone_name]
		);

		let timezoneNameEntries = [
			...Object.keys($timezones).map(tz=>[tz, tz]), 
			...countryToTimezoneMap
		];

		let timezoneNames = Object.fromEntries(timezoneNameEntries);
		return timezoneNames;
	});



	const defaultTimezone = Intl.DateTimeFormat().resolvedOptions().timeZone;

	let selectedTimezones = [defaultTimezone];

	let anchorTime = Date.now();

	let timeRange = writable([anchorTime, anchorTime + 3 * 24 * 60 * 60 * 1000]);

	/**
	 * Offset the time range by a given offset in milliseconds.
	 * @param {number} offset
	 */
	function offsetTimeRange(offset) {
		$timeRange = $timeRange.map(t=>t + offset);
	}

	const day = 24 * 60 * 60 * 1000;

	let hover = writable(null);
	let selection = writable(null);

	let animate = writable(true);

	// Function to copy the start of the selected time range
	// For all the timezones, to the clipboard
	function copyStart() {
		if($selection === null) return;

		/**
		 * @param {number} time
		 * @param {string} timezone
		 * @returns {string}
		 */
		function getTimeString(time, timezone) {
			return Intl.DateTimeFormat(undefined, {
				weekday: 'short',
				month: 'short',
				day: 'numeric',
				hour: 'numeric',
				hour12: true,
				minute: 'numeric',
				timeZone: timezone
			}).format(time);
		}

		// @ts-ignore
		let localTimes = selectedTimezones.map(tz=>tz + ": " + getTimeString($selection[0], tz));

		navigator.clipboard.writeText(localTimes.join(", "));
	}
</script>

<svelte:head>
    <title>Cross-Timezone Events</title> 
</svelte:head>

A simple way to coordinate cross-timezone events.

<p>
	<button on:click={() => offsetTimeRange(-7 * day)}>&lt;&lt;</button>
	<button on:click={() => offsetTimeRange(-day)}>&lt;</button>
	{new Date($timeRange[0]).toLocaleString()}
	<button on:click={() => offsetTimeRange(day)}>&gt;</button>
	<button on:click={() => offsetTimeRange(7 * day)}>&gt;&gt;</button>
</p>
<p>
	{#each selectedTimezones as timezone}
		<Timezone {timezoneNames} {timezones} {cities} {anchorTime} {timeRange} {hover} {selection} {animate} bind:value={timezone} />
	{/each}
	<button on:click={() => selectedTimezones = [...selectedTimezones, defaultTimezone]}>Add Timezone</button>
</p>

<p>
	<button on:click={copyStart}>Copy Start Times</button>
</p>

<p>
Cities data from <a href="https://simplemaps.com/data/world-cities">https://simplemaps.com/data/world-cities</a>.<br/>
Time zones from <a href="https://www.iana.org">https://www.iana.org</a> processed by <a href="https://timezonedb.com">https://timezonedb.com</a>
</p>