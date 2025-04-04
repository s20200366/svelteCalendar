<script>
    import {getContext} from 'svelte';
    import {
        cloneDate,
        addDuration,
        datesEqual,
        createEventChunk,
        eventIntersects,
        floor,
        rect,
        setPayload
    } from '@event-calendar/core';
    import {groupEventChunks} from './utils';
    import Event from './Event.svelte';
    import NowIndicator from './NowIndicator.svelte';

    export let date;
    export let resource = undefined;

    let {_events, _iEvents, highlightedDates, nowIndicator, slotDuration, slotHeight, theme,
        _interaction, _today, _slotTimeLimits} = getContext('state');

    let el;
    let chunks, bgChunks, iChunks = [];
    let isToday, highlight;

    let start, end;

    $: {
        start = addDuration(cloneDate(date), $_slotTimeLimits.min);
        end = addDuration(cloneDate(date), $_slotTimeLimits.max);
    }

    $: {
        chunks = [];
        bgChunks = [];
        for (let event of $_events) {
            if (!event.allDay && eventIntersects(event, start, end, resource, true)) {
                let chunk = createEventChunk(event, start, end);
                switch (event.display) {
                    case 'background': bgChunks.push(chunk); break;
                    default: chunks.push(chunk);
                }
            }
        }
        groupEventChunks(chunks);
    }

    $: iChunks = $_iEvents.map(
        event => event && eventIntersects(event, start, end, resource, true) ? createEventChunk(event, start, end) : null
    );

    $: isToday = datesEqual(date, $_today);
    $: highlight = $highlightedDates.some(d => datesEqual(d, date));

    function dateFromPoint(y) {
        y -= rect(el).top;
        return {
            allDay: false,
            date: addDuration(
                addDuration(cloneDate(date), $_slotTimeLimits.min),
                $slotDuration,
                floor(y / $slotHeight)
            ),
            resource,
            dayEl: el
        };
    }

    $: if (el) {
        setPayload(el, dateFromPoint);
    }

    function createPointerEnterHandler(interaction) {
        return interaction.pointer
            ? jsEvent => interaction.pointer.enterTimeGrid(date, el, jsEvent, resource)
            : undefined;
    }
</script>

<div
    bind:this={el}
    class="{$theme.day} {$theme.weekdays[date.getUTCDay()]}{isToday ? ' ' + $theme.today : ''}{highlight ? ' ' + $theme.highlight : ''}"
    on:pointerenter={createPointerEnterHandler($_interaction)}
    on:pointerleave={$_interaction.pointer?.leave}
    on:pointerdown={$_interaction.action?.select}
>
    <div class="{$theme.bgEvents}">
        {#each bgChunks as chunk (chunk.event)}
            <Event {date} {chunk}/>
        {/each}
    </div>
    <div class="{$theme.events}">
        <!-- Pointer -->
        {#if iChunks[1]}
            <Event {date} chunk={iChunks[1]}/>
        {/if}
        {#each chunks as chunk (chunk.event)}
            <Event {date} {chunk}/>
        {/each}
        <!-- Drag, Resize & Select -->
        {#if iChunks[0] && !iChunks[0].event.allDay}
            <Event {date} chunk={iChunks[0]}/>
        {/if}
    </div>
    <div class="{$theme.extra}">
        <!-- Now indicator -->
        {#if $nowIndicator && isToday}
            <NowIndicator />
        {/if}
    </div>
</div>