<script>
	import * as d3 from 'd3';
	import { onMount } from 'svelte';
	import { readable, writable } from "svelte/store";

    import SunCalc from 'suncalc';
	import { crossfade, draw, slide } from 'svelte/transition';
	import { text } from '@sveltejs/kit';

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
    export let anchorTime = Date.now();

    export let timeRange = writable([anchorTime, anchorTime + 3 * 1000 * 60 * 60 * 36]);

    $: timezoneTimeRange = $timeRange.map(t => t + timezoneOffset);

    let userTimezoneOffset = new Date().getTimezoneOffset() * 60 * 1000;

    let x = d3.scaleTime()
    $: {
        let _domain = [anchorTime, timezoneTimeRange[1] + anchorTime - timezoneTimeRange[0]];
        x.domain(_domain);
        x = x;
    }
    
    
    
    let y = d3.scaleLinear()
        .domain([0,1])

    /** @type {SVGSVGElement} */
    let svg;

    function onresize() {
        if(svg) {
            let yMargin = 12;
            x.range([0, svg.clientWidth]);
            y.range([yMargin, svg.clientHeight-yMargin]);
        }
        x = x;
        y = y;
        sunY = sunY;
    }

    let animate = false;
    onMount(() => {
        window.addEventListener("resize", onresize);
        onresize();

        // Only start animation after the first render
        setTimeout(() => {
            animate = true;
        }, 0);

        return () => {
            window.removeEventListener("resize", onresize);
        }
    });


    let now = Date.now();
    $: timezoneOffset = getTimezoneOffset(now, value);




    /**
     * @param {Date|number} time
     * @param {string} timezone
     */
     function getTimezoneOffset(time, timezone) {
        let _time = time instanceof Date ? time.getTime() : time;

        let offsetInSeconds = $timezones[timezone]?.find(tz => tz.time_start <= _time)?.gmt_offset;

        return offsetInSeconds ?? 0;
    }

    /**
     * All local timezones that overlap with the timeRange
     * @type {Timezone[]}
     */
    let localTimezones = [];
    $: {
        localTimezones = [];
        let tz = $timezones[value];
        
        if(tz) {
            // Sort by descending start time
            tz = tz.filter(t=>t.time_start < $timeRange[1])
                .sort((a,b) => b.time_start - a.time_start);

            let i = 0;
            while((i < tz.length) && (tz[i].time_start > $timeRange[0])) {
                localTimezones.push(tz[i]);
                i++;
            }
            // Add one more timezone to cover the entire timeRange
            if(i < tz.length) {
                localTimezones.push(tz[i]);
            }
        }

        // Sort in ascending order
        localTimezones = localTimezones.sort((a,b) => a.time_start - b.time_start);
    }

    $: localTimezoneHourTicks = localTimezones.map((tz, i, a) => {
        const hour = 1000 * 60 * 60;
        const day = 1000 * 60 * 60 * 24;

        let ticksPerDay = 24;
        let tickInterval = day / ticksPerDay;

        let start = Math.max($timeRange[0], tz.time_start);
        start += tz.gmt_offset;
        start = Math.floor(start / tickInterval) * tickInterval;

        let end = $timeRange[1];
        if(i < a.length - 1) {
            end = Math.min(end, a[i+1].time_start-1);
        }
        end += tz.gmt_offset;
        end = Math.floor(end / tickInterval) * tickInterval;
        
        /** @type {{h:number, day?:string}[]}*/
        let ticks = [];
        for(let h = start; h < end + tickInterval; h += tickInterval) {
            /** @type {{h:number, day?:string}} */
            let tick = {h};
            if(h % day == 0) {
                tick.day = formatDate(h - tz.gmt_offset);
            }
            ticks.push(tick);
        }
        return ticks;
    });



    // Update now every second
    setInterval(() => {
        now = Date.now();
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
     * An array of ticks for each hour across 3 days, centred on the anchor time
     * @type {number[]}
     */
     let hourTicks = [];
    $: {
        hourTicks = [];
        const hour = 1000 * 60 * 60;
        let start = Math.floor(timezoneTimeRange[0] / hour) * hour;
        for(let h = start; h < timezoneTimeRange[1] + hour; h += hour) {
            hourTicks.push(h);
        }
    }

    /**
     * Array of days where it's midnight in the selected timezone
     * Calculated from x.domain()
     * @type {number[]}
     */
    let dayTicks = [];
    $: {
        dayTicks = [];

        const day = 1000 * 60 * 60 * 24;
        let start = timezoneTimeRange[0];
        start = Math.floor((start - getTimezoneOffset(start, value)) / day) * day;
        
        for(let d = start; d < timezoneTimeRange[1] + day; d += day) {
            dayTicks.push(d);
        }
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
     * @typedef {[number, number|null]} SunElevation
     */

    /**
     * Array of sun elevation angles based on hourTicks
     * @type {SunElevation[]}
     */
    let sunElevation = [];
    
    $: {
        sunElevation = [];
        if(location) {
            // Get the sunrise and sunset times between the first and last hour ticks
            const day = 1000 * 60 * 60 * 24;
            for(let d = timezoneTimeRange[0]; d < timezoneTimeRange[1] + day; d += day) {
                let sunTimes = SunCalc.getTimes(d, location[0], location[1]);
                // @ts-ignore sunrise exists
                let sunrise = sunTimes.sunrise.getTime();
                // @ts-ignore sunset exists
                let sunset = sunTimes.sunset.getTime();
                let noon = sunTimes.solarNoon.getTime();

                sunElevation.push([sunrise-1, null]);
                // Get the sun elevation for each half hour block
                const blockInMins = 15;
                for(let t = sunrise; t < sunset; t += 1000 * 60 * blockInMins) {
                    let sunPos = SunCalc.getPosition(t, location[0], location[1]);
                    sunElevation.push([t, sunPos.altitude]);

                    if(t <= noon && noon <= t + 1000 * 60 * blockInMins) {
                        let sunPos = SunCalc.getPosition(new Date(noon), location[0], location[1]);
                        sunElevation.push([noon, sunPos.altitude]);
                    }
                }
                let t= sunset;
                let sunPos = SunCalc.getPosition(t, location[0], location[1]);
                sunElevation.push([t, sunPos.altitude]);

                sunElevation.push([sunset+1, null]);
            }
        }
    }

    let sunY = d3.scaleLinear()
    $: {
        sunY.domain([d3.min(sunElevation, (/** @type {SunElevation} */ d) => d[1]), Math.PI/2]);
        sunY.range([y(1), y(0)]);
        sunY = sunY;
    }

    let sunArea = d3.area()
    $:{
        sunArea
            .x((/** @type {SunElevation} */ d) => x(d[0] + getTimezoneOffset(d[0], value)))
            .y0(svg?.clientHeight || 10)
            .y1((/** @type {SunElevation} */ d) => sunY(d[1]))
            .defined((/** @type {SunElevation} */ d) => d[1] !== null);
        sunArea = sunArea;
    }

    /** @type {(date:number)=>boolean} */
    $: isMidnight = (date) => {
        return dayTicks.includes(date);
    }

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
    <svg class={`${animate ? "animate" : ""}`} bind:this={svg} height="3em">
        <defs>
            <linearGradient id="sun-gradient" x1="0%" y1="0%" x2="0%" y2="100%">
                <stop offset="0%" stop-color="rgb(255, 204, 0)" stop-opacity="0.3" />
                <stop offset="100%" stop-color="rgb(255, 204, 0)" stop-opacity="0.05" />
            </linearGradient>
        </defs>
        <g class={`timeline`} style={`transform: translate(${x(anchorTime) - x($timeRange[0])}px)`}>
            <g class="localTime" style={`transform: translate(${x(0) - x(timezoneOffset)}px)`}>
                <g class="sun">
                    <path class="area" d={sunArea(sunElevation)} />
                    <path class="outline" d={sunArea.lineY1()(sunElevation)} />
                </g>
            </g>
            {#each localTimezones as tz}
                <g class={`ticks local ${tz.dst ? "dst":""}`} style={`transform: translate(${x(0) - x(tz.gmt_offset)}px)`}>
                    {#each localTimezoneHourTicks[localTimezones.indexOf(tz)] as tick (tick)}
                        <line class={`${tick.day ? "midnight" : ""}`} x1={x(tick.h)} x2={x(tick.h)} y1={y(0)} y2={y(1)} />
                        {#if tick.day}
                            <text x={x(tick.h)} y={y(1)} dy="1em" text-anchor="start">{tick.day}</text>
                        {/if}
                    {/each}
                </g>
            {/each}
            <g class="now">
                <line x1={x(now)} x2={x(now)} y1={y(0)} y2={y(1)} />
                <text x={x(now)} y={y(0)} dy={-2} text-anchor="start">{formatTime(now)}</text>
                <text x={x(now)} y={y(0)} dx="-0.3em" dy={-2} text-anchor="end">{formatDate(now)}</text>
            </g>
        </g>
    </svg>
</div>

<style lang="scss">
    select {
        width: 100px;
    }
    input[type="search"] {
        width: 100px;
    }

    div.timezone {
        display: flex;
        flex-flow: row wrap;
        align-items: center;

        margin-bottom: 0.3em;

        gap: 0.5em;
        
        & > div {
            flex: 0 0 auto;
        }

        & > svg {
            flex: 1 0 300px;

        }
    }

    svg {
        user-select: none;

        text {
            font-size: 10px;
            font-family: sans-serif;
        }
    }

    svg.animate {
        g.timeline, g.localTime, g.local {
            transition: transform 0.3s ease-in-out;
        }
    }

    g.timeline {
        cursor: cell;
    }

    g.sun {
        --color-sun: rgb(255, 204, 0);

        .area {
            fill: url(#sun-gradient);
            stroke: none;
        }

        .outline {
            stroke: var(--color-sun);
            stroke-width: 2px;

            fill: none;
        }
    }

    g.ticks {
        &.local {
            line {
                stroke: #00aaff88;
                stroke-width: 1px;

                &.midnight {
                    stroke-width: 2px;
                }
            }

            &.dst line {
            stroke: #ffae0088;
        }

        }
        
        text {
            fill: #9999;
        }
    }

    g.now {
        --color-now: #f00;

        line {
            stroke: var(--color-now);

            transition: all 0.3s;
        }

        text {
            fill: var(--color-now);

            transition: x 0.3s;
        }
    }
</style>