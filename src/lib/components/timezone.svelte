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

    /** @type {import('svelte/store').Writable<number[]>} */
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

    /** @type {number} */
    let resizeTimeout;
    function onresize() {
        if(svg) {
            $animate = false;

            let yMargin = 16*1.2;
            x.range([0, svg.clientWidth]);
            y.range([yMargin, svg.clientHeight-yMargin]);
        }
        x = x;
        y = y;
        sunY = sunY;

        clearTimeout(resizeTimeout);
        resizeTimeout = setTimeout(() => {
            $animate = true;
        }, 100);
    }

    export let animate = writable(true);

    let _initalised = false;
    let _animate = false;
    onMount(() => {
        window.addEventListener("resize", onresize);
        onresize();

        // Only start animation after the first render
        requestAnimationFrame(() => {
            _initalised = true;
            _animate = $animate;
        });

        if(searchInput) {
            searchInput.focus();
        }

        return () => {
            window.removeEventListener("resize", onresize);
        }
    });

    $: if(_initalised) _animate = $animate;


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

        if(i == 0) {
            let dayStart = Math.floor(start / day) * day;
            if(dayStart < start)
                ticks.push({h: dayStart, day: formatDate(dayStart - tz.gmt_offset)});
        }

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

    /**
     * Identify the work hours for each local timezone
     * @type {[number,number][][]}
     */
    $: localTimezoneOffWorkHours = localTimezones.map((tz, i, a) => {
        const hour = 1000 * 60 * 60;
        const day = 1000 * 60 * 60 * 24;

        let workHours = [8, 18];
        workHours = workHours.map(h => h * hour);
        
        let start = Math.max($timeRange[0], tz.time_start);
        start += tz.gmt_offset;
        start = Math.floor(start / day) * day;
        if(i > 0) {
            start = a[i].time_start + tz.gmt_offset;
        }

        let end = $timeRange[1];
        end += tz.gmt_offset;
        end = Math.ceil(end / day) * day;
        if(i < a.length - 1) {
            end = a[i+1].time_start + tz.gmt_offset;
        }

        /** @type {[number,number][]} */
        let blocks = [];

        while(start <= end) {
            let dayStart = Math.floor(start / day) * day;
            let workStart = dayStart + workHours[0];
            let workEnd = dayStart + workHours[1];

            if(workStart > start) {
                blocks.push([Math.max(start, dayStart), Math.min(end, workStart)]);
            }
            if(workEnd < end) {
                blocks.push([Math.max(start, workEnd), Math.min(end, dayStart + day)]);
            }

            start = dayStart + day;
        }

        return blocks;
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

    $: formatDateTime = Intl.DateTimeFormat(undefined, {
        weekday: 'short',
        month: 'short',
        day: 'numeric',
        hour: 'numeric',
        minute: 'numeric',
        timeZone: value
    }).format;

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

            let start = $timeRange[0] + getTimezoneOffset($timeRange[0], value);
            start = Math.floor(start / day) * day;
            start -= day;

            let end = $timeRange[1] + getTimezoneOffset($timeRange[1], value);
            end = Math.ceil(end / day) * day;
            end += day;

            let d = start;

            while(d <= end) {
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

                d += day;
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

    /**
     * Svelte Transition function to delay the disappearance of the hour marks when the time range changes
     * @param {SVGElement} node
     * @param {{duration: number}} param
     * 
    */
    function delay(node, {duration}) {
        return {
            duration,
            
        }
    }

    /**
     * @type {import('svelte/store').Writable<null|[number,number]>}
     */
    export let selection = writable(null);

    /**
     * @type {import('svelte/store').Writable<null|number>}
     */
     export let hover = writable(null);

    const minSelectionBlock = 1000 * 60 * 15;

    /**
     * Update the hover store with the hovered time
     * @param {MouseEvent} event
     */
    function onHover(event) {
        let time = timeFromClientX(event.clientX);

        // Snap to nearest 15 min block
        time = Math.round(time / minSelectionBlock) * minSelectionBlock;

        $hover = time;

        onUpdateSelection($hover);
    }


    /** @type {number|undefined} */
    let selectionStart;

    /** 
     * @param {number|null} time
     */
    function onStartSelection(time) {
        if(!time) return;
        selectionStart = time;
        $selection = [selectionStart, selectionStart + minSelectionBlock];
    }

    /** 
     * @param {number|null} time
     */
    function onUpdateSelection(time) {
        if(selectionStart && time) {
            if(time == selectionStart) {
                $selection = [time, selectionStart + minSelectionBlock];
            } else if(time < selectionStart) {
                $selection = [time, selectionStart];
            } else {
                $selection = [selectionStart, time];
            }
        }
    }

    function onEndSelection() {
        selectionStart = undefined;
    }


    /** @type {number|undefined} */
    let scrollTimeout;

    /**
     * @type {import('svelte/elements').UIEventHandler<SVGRectElement>}
     */
    let onScroll = (event)=> {
        // Zoom in or out based on the scroll direction
        // With the zoom centred on the mouse position

        // @ts-ignore
        let eventX = event.clientX - event.target?.getBoundingClientRect().left;
        let time = x.invert(eventX + x($timeRange[0]) - x(anchorTime));

        if(time instanceof Date) {
            time = time.getTime();
        }

        // @ts-ignore
        let delta = -event.deltaY;
        let zoomFactor = 1.1;
        if(delta > 0) {
            zoomFactor = 1/zoomFactor;
        }

        // Clamp the zoomFactor such that the time range is at least 1 day and at most 14 days
        let minTimeRange = 1000 * 60 * 60 * 24;
        let maxTimeRange = 1000 * 60 * 60 * 24 * 14;
        let rangeDuration = $timeRange[1] - $timeRange[0];
        if(rangeDuration * zoomFactor < minTimeRange) {
            zoomFactor = minTimeRange / rangeDuration;
        } else if(rangeDuration * zoomFactor > maxTimeRange) {
            zoomFactor = maxTimeRange / rangeDuration;
        }


        let newTimeRange = $timeRange.map(t => time + (t - time) * zoomFactor);
        $timeRange = newTimeRange;

        $animate = false;
        clearTimeout(scrollTimeout);
        scrollTimeout = setTimeout(() => {
            $animate = true;
        }, 500);
    }

    $: duration = $animate ? 300 : 0;

    /**
     * @type {(time: number) => string}
     */
    $: translateX = (time) =>{
        return `transform: translate(${x(time)}px)`
    }

    /** @type {HTMLInputElement} */
    let searchInput;


    /**
     * @param {number} clientX
     */
    function timeFromClientX(clientX) {
        // @ts-ignore
        let eventX = clientX - svg.getBoundingClientRect().left;
        let time = x.invert(eventX + x($timeRange[0]) - x(anchorTime));

        if(time instanceof Date) {
            time = time.getTime();
        }

        return time;
    }

    /** @type {number|undefined} */
    let touchSelectionHoldTimeout;

    /** @type {{timeRange:number[], clientX:number, zoomFactor:number} | undefined} */
    let touchStart;

    /**
     * @param {TouchEvent} event
     */
    function touchstart(event) {
        event.preventDefault();

        let touch = event.touches[0];
        let time = timeFromClientX(touch.clientX);
        time = Math.round(time / minSelectionBlock) * minSelectionBlock;

        touchStart = {
            timeRange: $timeRange, 
            clientX: touch.clientX, 
            zoomFactor: ($timeRange[1] - $timeRange[0]) / (x.range()[1] - x.range()[0])
        };

        clearTimeout(touchSelectionHoldTimeout);
        touchSelectionHoldTimeout = setTimeout(() => {
            onStartSelection(time);
        }, 500);
    }

    /** @type {number|undefined} */
    let secondTouchStart;

    let zoomX = d3.scaleLinear();

    /**
     * @param {TouchEvent} event
    */
    function touchmove(event) {
        event.preventDefault();

        clearTimeout(touchSelectionHoldTimeout);

        if(selectionStart) {
            let touch = event.touches[0];
            let time = timeFromClientX(touch.clientX);
            time = Math.round(time / minSelectionBlock) * minSelectionBlock;

            onUpdateSelection(time);
        } else if(touchStart) {
            $animate = false;

            // Drag the timeline
            let touch = event.touches[0];
            let offset = timeFromClientX(touch.clientX) - timeFromClientX(touchStart.clientX);
            $timeRange = touchStart.timeRange.map(t => t - offset);
            
            if(event.touches.length > 1) {
                if(!secondTouchStart) {
                    secondTouchStart = event.touches[1].clientX;
                    zoomX.range([touch.clientX, secondTouchStart].map(x => x - svg.getBoundingClientRect().left));
                    zoomX.domain([timeFromClientX(touch.clientX), timeFromClientX(secondTouchStart)]);
                } else {
                    zoomX.range([touch.clientX, event.touches[1].clientX].map(x => x - svg.getBoundingClientRect().left));

                    let newTimeRange = x.range().map(r => zoomX.invert(r));
                    console.log(newTimeRange.map(t => formatDateTime(t)));
                    
                    // Clamp the zoomFactor such that the time range is at least 1 day and at most 14 days
                    let minTimeRange = 1000 * 60 * 60 * 24;
                    let maxTimeRange = 1000 * 60 * 60 * 24 * 14;
                    let rangeDuration = Math.abs(newTimeRange[1] - newTimeRange[0]);
                    if(rangeDuration < minTimeRange) {
                        // Rescale the time range centred on touch
                        let time = zoomX.invert((touch.clientX + event.touches[1].clientX) / 2);
                        let zoomFactor = minTimeRange / rangeDuration;
                        newTimeRange = newTimeRange.map(t => time + (t - time) * zoomFactor);
                        
                    } else if(rangeDuration > maxTimeRange) {
                    }

                    // A scale of 0 to 1 where 0 is the min time range and 1 is the max time range
                    let rangeDurationScale = (rangeDuration - minTimeRange) / (maxTimeRange - minTimeRange);

                    console.log(rangeDurationScale);

                    $timeRange = newTimeRange;
                }

            } else {
                if(secondTouchStart) {
                    touchStart.timeRange = $timeRange;
                }
                secondTouchStart = undefined;
            }
        }
    }

    /**
     * @param {TouchEvent} event
     */
    function touchend(event) {
        event.preventDefault();

        clearTimeout(touchSelectionHoldTimeout);

        if(selectionStart) {
            onEndSelection();
        } else {
            $animate = true;
        }
    
    }
</script>

<div class="timezone">
    <div>
        <input type="search" class="search" bind:this={searchInput} bind:value={searchString} placeholder="Search" on:keydown={onSearchStringChange} />
        <select class="timezone" bind:value={dropdownValue}>
            {#each timezoneSearchResults as timezoneName}
                <option value={timezoneName}>{timezoneName}</option>
            {/each}
        </select>
    </div>
    <svg class={`${_animate ? "animate" : ""}`} bind:this={svg} height="4em">
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
                    {#each localTimezoneHourTicks[localTimezones.indexOf(tz)] as tick (tick.h)}
                        <line out:delay={{duration}} class={`${tick.day ? "midnight" : ""}`} x1={x(tick.h)} x2={x(tick.h)} y1={y(0)} y2={y(1)} />
                        {#if tick.day}
                            <text out:delay={{duration}} x={x(tick.h)} y={y(1)} dy="1em" text-anchor="start">{tick.day}</text>
                        {/if}
                    {/each}
                    {#each localTimezoneOffWorkHours[localTimezones.indexOf(tz)] as block (block[0])}
                        <rect class="offWork" out:delay={{duration}} x={x(block[0])} y={y(0)} width={x(block[1]) - x(block[0])} height={y(1) - y(0)} fill="#f00" fill-opacity="0.1" />
                    {/each}
                </g>
            {/each}
            <g class="now" style={translateX(now)}>
                <line x1={0} x2={0} y1={y(0)} y2={y(1)} />
                <text x={0} y={y(0)} dy={-2} text-anchor="start">{formatTime(now)}</text>
                <text x={0} y={y(0)} dx="-0.3em" dy={-2} text-anchor="end">{formatDate(now)}</text>
            </g>

            {#if $selection}
            <g class="selection">
                <rect x={x($selection[0])} y={y(0)} width={x($selection[1]) - x($selection[0])} height={y(1) - y(0)} />
                <text x={x($selection[0])} y={y(0)} dy={-2} text-anchor="end">{formatDateTime($selection[0])}</text>
                <text x={x($selection[1])} y={y(0)} dy={-2} text-anchor="start">{formatDateTime($selection[1])}</text>
            </g>
            {/if}


            {#if $hover}
                <g class="hover">
                    <rect class="backing" x={x($hover)} y={y(0)-15} width="100" height={15} />
                    <text x={x($hover)} y={y(0)} dy={-2} text-anchor="start">{formatDateTime($hover)}</text>
                    <rect class="marker" x={x($hover)} y={y(0)} width="1" height={y(1) - y(0)} />
                </g>
            {/if}
        </g>

        <!-- svelte-ignore a11y-no-noninteractive-element-interactions -->
        <rect class="hitarea" x={x.range()[0]} y={y.range()[0]} width={Math.abs(x.range()[1] - x.range()[0])} height={Math.abs(y.range()[1] - y.range()[0])} fill="transparent" stroke="none" 
            on:mousemove={onHover}
            on:mouseleave={() => {$hover = null; onEndSelection()}}
            on:mousedown={()=>onStartSelection($hover)}
            on:mouseup={onEndSelection}
            on:wheel={onScroll}
            on:touchstart={touchstart}
            on:touchmove={touchmove}
            on:touchend={touchend}
            on:touchcancel={touchend}
            role="application"
            tabindex="-1"
        />
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
            font-size: 14px;
            font-family: sans-serif;
        }

        .hitarea {
            cursor: cell;
            outline: none;
        }

        .selection {
            fill: #00aaff;
            stroke: none;

            rect {
                opacity: 0.3;
            }
        }
    }

    svg.animate {
        --duration: 0.3s;

        g.timeline, g.localTime, g.local {
            transition: transform var(--duration) ease-in-out;
        }
    }

    g.hover {
        .backing {
            fill: #fff9;
            stroke: none;
        }

        .marker {
            fill: #000;
        }
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
                stroke: #a4a4a488;
                stroke-width: 1px;

                &.midnight {
                    stroke-width: 2px;
                }
            }

            &.dst line {
                stroke: #de980088;
            }

            .offWork {
                fill: #0009;
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
        }

        text {
            fill: var(--color-now);
        }
    }
</style>