<script>
	import * as d3 from 'd3';
	import { onMount } from 'svelte';
	import { readable } from "svelte/store";


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

        return $timezones[timezone]?.find(tz => tz.time_start <= _time)?.gmt_offset ?? 0;
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
        <g class="ticks">
            {#each hourTicks as tick}
                <line x1={x(tick)} x2={x(tick)} y1={y(0)} y2={y(1)} />
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

    g.ticks {
        stroke: #999;
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