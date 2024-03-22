<script>
	import * as d3 from 'd3';
	import { onMount } from 'svelte';
	import { readable } from "svelte/store";

    import SunCalc from 'suncalc';

	/**
	 * zone_name,country_code,abbreviation,time_start,gmt_offset,dst
	 * @typedef {Object} Timezone
	 * @property {string} zone_name
	 * @property {string} country_code
	 * @property {string} abbreviation
	 * @property {EpochTimeStamp} time_start 
	 * @property {number} gmt_offset in seconds
	 * @property {boolean} dst Whether daylight savings is in effect
	 */

    /** @type {import("svelte/store").Readable<Record<string, string>>} */
    export let timezoneNames;

    /** @type {import("svelte/store").Readable<Record<string, Timezone[]>>} */
    export let timezones;


    export let value = "";
    $: value = $timezoneNames[dropdownValue];

    let searchString = "";

    export let dropdownValue = value;

    $: timezoneSearchResults = Object.keys($timezoneNames).filter(name => searchString.trim() == "" ? true : name.toLowerCase().includes(searchString.trim().toLowerCase()));

    /** 
     * @param {KeyboardEvent} event
     */
    function onSearchStringChange(event) {
        if(!timezoneSearchResults.includes(dropdownValue)) {
            dropdownValue = timezoneSearchResults[0];
        }
    }

    let userTimezoneOffset = new Date().getTimezoneOffset() * 60;
    /** 
     * An array of ticks for each hour across 3 days, centred on the current time
     * @type {number[]}
     */
    export let hourTicks = Array.from({ length: 72 }, (_, i) => {
        let now = Date.now() - userTimezoneOffset;
        // Round this off to the nearest hour
        now = now - now % (1000 * 60 * 60);
        let hour = 1000 * 60 * 60;
        let offset = (i - 36) * hour;
        return now + offset;
    });

    let x = d3.scaleTime()

    $: x.domain(d3.extent(hourTicks));
    
    let y = d3.scaleLinear()
        .domain([0,1])

    /** @type {SVGSVGElement} */
    let svg;

    function onresize() {
        if(svg) {
            x.range([0, svg.clientWidth]);
            y.range([10, svg.clientHeight-10]);
        }
        x = x;
        y = y;
        sunY = sunY;
    }

    onMount(() => {
        window.addEventListener("resize", onresize);
        onresize();

        return () => {
            window.removeEventListener("resize", onresize);
        }
    });


    $: now = Date.now();
    $: timezoneOffset = getTimezoneOffset(now, dropdownValue);



    /**
     * @param {Date|number} time
     * @param {string} timezone
     */
     function getTimezoneOffset(time, timezone) {
        let _time = time instanceof Date ? time.getTime() : time;

        let offsetInSeconds = $timezones[timezone]?.find(tz => tz.time_start <= _time)?.gmt_offset;

        let tz = $timezones[timezone]?.find(tz => tz.time_start <= _time);

        return offsetInSeconds ?? 0;
    }


    /**
     * @param {Date|number} time
     * @param {number} timezoneOffset
     * @returns {number}
     */
    function addTimezoneOffset(time, timezoneOffset) {
        let _time = time instanceof Date ? time.getTime() : time;
        return _time;
    }

    // Update now every second
    setInterval(() => {
        // now = Date.now();
    }, 1000);

    $: formatTime = Intl.DateTimeFormat(undefined, {
        hour: 'numeric',
        minute: 'numeric',
        timeZone: value
    }).format;

    $: formatDate = Intl.DateTimeFormat(undefined, {
        weekday: 'short',
        month: 'short',
        day: 'numeric',
        timeZone: value
    }).format;


    /**
     * Array of days where it's midnight in the selected timezone
     * Calculated from x.domain()
     * @type {number[]}
     */
    let dayTicks = [];
    $: {
        let start = x.domain()[0].getTime() + timezoneOffset;
        let end = x.domain()[1].getTime() + timezoneOffset;
        let day = 1000 * 60 * 60 * 24;
        dayTicks = Array.from({ length: Math.ceil((end - start) / day) + 1 }, (_, i) => start - (start % day) + i * day - timezoneOffset);
    }



    export let cities = readable([]);

    /** @type {[number, number] | undefined} */
    let location;

    // Get the lat-long for the selected timezone
    $: {
        // First try to match the country name to the dropdown value
        let city = $cities.find(city => city.capital == 'primary' &&  city.country.toLowerCase() == dropdownValue?.toLowerCase());

        if(!city) {
            // If that doesn't work, the dropdown value could be a timezone name, let's find a city from there
            let possibleCityName = dropdownValue?.split("/")[1];
            // Replace underscores with spaces
            possibleCityName = possibleCityName?.replace(/_/g, " ");

            city = $cities.find(city => city.city_ascii.toLowerCase() == possibleCityName?.toLowerCase());
        }

        if(city) {
            location = [city.lat, city.lng];
        } else {
            location = undefined;
        }
    }

    /**
     * Array of sun elevation angles based on hourTicks
     * @type {[number,number|null][]}
     */
    let sunElevation = [];
    
    $: {
        sunElevation = [];
        if(location) {
            // Get the sunrise and sunset times between the first and last hour ticks
            for(let day of dayTicks) {
                let sunTimes = SunCalc.getTimes(new Date(day), location[0], location[1]);
                // @ts-ignore sunrise exists
                let sunrise = sunTimes.sunrise.getTime();
                // @ts-ignore sunset exists
                let sunset = sunTimes.sunset.getTime();
                let noon = sunTimes.solarNoon.getTime();

                sunElevation.push([sunrise-1, null]);
                // Get the sun elevation for each half hour block
                const blockInMins = 15;
                for(let t = sunrise; t < sunset; t += 1000 * 60 * blockInMins) {
                    let sunPos = SunCalc.getPosition(new Date(t), location[0], location[1]);
                    sunElevation.push([t, sunPos.altitude]);

                    if(t <= noon && noon <= t + 1000 * 60 * blockInMins) {
                        let sunPos = SunCalc.getPosition(new Date(noon), location[0], location[1]);
                        sunElevation.push([noon, sunPos.altitude]);
                    }
                }
                let t= sunset;
                let sunPos = SunCalc.getPosition(new Date(t), location[0], location[1]);
                sunElevation.push([t, sunPos.altitude]);

                sunElevation.push([sunset+1, null]);
            }
        }
    }

    let sunY = d3.scaleLinear()

    $: sunY.domain([d3.min(sunElevation, d => d[1]), Math.PI/2]);

    $: sunY.range([y(1), y(0)]);

    $: sunLine = d3.line()
        .x((d, i) => x(d[0]))
        .y(d => sunY(d[1]))
        .defined(d => d[1] !== null);

    
</script>

<div class="timezone">
    <div>
        <input type="search" class="search" bind:value={searchString} placeholder="Search" on:keydown={onSearchStringChange} />
        <select class="timezone" bind:value={dropdownValue}>
            {#each timezoneSearchResults as timezoneName}
                <option value={timezoneName}>{timezoneName}</option>
            {/each}
        </select>
    </div>
    <svg bind:this={svg} height="3em">
        <g class="sun">
            <path d={sunLine(sunElevation)} />
        </g>
        <g class="ticks">
            {#each hourTicks as tick}
                <line x1={x(tick)} x2={x(tick)} y1={y(0)} y2={y(1)} />
            {/each}
            {#each dayTicks as tick}
                <text x={x(tick)} y={y(1)} dy="1em" text-anchor="start">{formatDate(tick)}</text>
            {/each}
        </g>
        <g class="now">
            <line x1={x(now)} x2={x(now)} y1={y(0)} y2={y(1)} />
            <text x={x(now)} y={y(0)} dy={-2} text-anchor="start">{formatTime(now)}</text>
            <text x={x(now)} y={y(0)} dx="-0.3em" dy={-2} text-anchor="end">{formatDate(now)}</text>
        </g>
    </svg>
</div>

<style lang="scss">
    select {
        width: 100px;
    }

    div.timezone {
        display: flex;
        flex-direction: row nowrap;
        align-items: center;

        margin-bottom: 0.3em;

        gap: 0.5em;
        
        & > div {
            flex-basis: 0 0;
        }

        & > svg {
            flex-grow: 1;
        }
    }

    svg {
        user-select: none;

        text {
            font-size: 10px;
            font-family: sans-serif;
        }
    }

    g.sun {
        --color-sun: rgb(255, 196, 0);

        path {
            stroke: var(--color-sun);
            stroke-width: 2px;

            fill: none;
        }
    }

    g.ticks {
        line {
            stroke: #ccc;
        }
        
        text {
            fill: #ccc;
        }
    }

    g.now {
        --color-now: #f00;

        line {
            stroke: var(--color-now);
        }

        text {
            fill: var(--color-now);
        }
    }
</style>