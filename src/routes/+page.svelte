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

	const day = 24 * 60 * 60 * 1000;	
	let timeRange = writable([anchorTime - .5*day, anchorTime + 2 * day]);

	/**
	 * Offset the time range by a given offset in milliseconds.
	 * @param {number} offset
	 */
	function offsetTimeRange(offset) {
		$timeRange = $timeRange.map(t=>t + offset);
	}


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

		console.log(localTimes.join(", "));

		navigator.clipboard.writeText(localTimes.join(", "));
	}


	function redirectToEventCreation() {
		if(!$selection) return;

		const baseUrl = 'https://calendar.google.com/calendar/r/eventedit';
		const params = new URLSearchParams();

		/**
		 * @param {number} dateTime
		 */
		function formatDate(dateTime) {
			return new Date(dateTime).toISOString().replace(/-|:|\.\d+/g, '');
		}

		params.append('dates', `${formatDate($selection[0])}/${formatDate($selection[1])}`);
		// Add more parameters as needed (e.g., location, description, etc.)

		const eventUrl = `${baseUrl}?${params.toString()}`;
		window.open(eventUrl);
	}

	function openICalEvent() {
		if(!$selection) return;

		function generateUniqueId() {
			return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function (c) {
				const r = (Math.random() * 16) | 0;
				const v = c === 'x' ? r : (r & 0x3) | 0x8;
				return v.toString(16);
			});
		}

		/**
		 * @param {Date|number} dateTime
		 * @param {boolean} [withTime=false]
		 */
		function formatDate(dateTime, withTime = false) {
			if (typeof dateTime === 'number') {
				dateTime = new Date(dateTime);
			}

			const year = dateTime.getUTCFullYear().toString();
			const month = (dateTime.getUTCMonth() + 1).toString().padStart(2, '0');
			const day = dateTime.getUTCDate().toString().padStart(2, '0');
			
			if (withTime) {
				const hours = dateTime.getUTCHours().toString().padStart(2, '0');
				const minutes = dateTime.getUTCMinutes().toString().padStart(2, '0');
				const seconds = dateTime.getUTCSeconds().toString().padStart(2, '0');
				return `${year}${month}${day}T${hours}${minutes}${seconds}Z`;
			} else {
				return `${year}${month}${day}`;
			}
		}

		const icalEvent = `BEGIN:VCALENDAR
VERSION:2.0
PRODID:-//Cross-Timezone Events//Event Creator//EN
BEGIN:VEVENT
UID:${generateUniqueId()}
DTSTAMP:${formatDate(Date.now(), true)}
DTSTART:${formatDate($selection[0], true)}
DTEND:${formatDate($selection[1], true)}
SUMMARY:New Event
END:VEVENT
END:VCALENDAR`;

		const blob = new Blob([icalEvent], { type: 'text/calendar;charset=utf-8' });
		const url = URL.createObjectURL(blob);
		const link = document.createElement('a');
		link.href = url;
		link.download = 'event.ics';
		link.click();
		URL.revokeObjectURL(url);
	}

</script>

<svelte:head>
    <title>Cross-Timezone Events</title> 
</svelte:head>

A simple way to coordinate cross-timezone events.

<p>
	<button on:click={() => offsetTimeRange(-7 * day)}>&lt;&lt;</button>
	<button on:click={() => offsetTimeRange(-day)}>&lt;</button>
	<button on:click={()=>$timeRange = [Date.now()-.5*day,Date.now()+2*day]}>Today</button>
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
	<button on:click={copyStart} disabled={!$selection}>Copy Start Times</button>
	<button on:click={redirectToEventCreation} disabled={!$selection}>Create Google Calendar Event</button>
	<button on:click={openICalEvent} disabled={!$selection}>Download iCal Event</button>
</p>

<p>
Cities data from <a href="https://simplemaps.com/data/world-cities">https://simplemaps.com/data/world-cities</a>.<br/>
Time zones from <a href="https://www.iana.org">https://www.iana.org</a> processed by <a href="https://timezonedb.com">https://timezonedb.com</a>
</p>