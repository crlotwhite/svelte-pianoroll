<!--
  Main PianoRoll component that integrates all subcomponents.
  This component serves as the container for the entire piano roll interface.
-->
<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import Toolbar from './Toolbar.svelte';
  import KeyboardComponent from './KeyboardComponent.svelte';
  import GridComponent from './GridComponent.svelte';
  import TimeLineComponent from './TimeLineComponent.svelte';
  
  // Props
  export let width = 1000;  // Total width of the piano roll
  export let height = 600;  // Total height of the piano roll
  export let keyboardWidth = 120; // Width of the keyboard component
  export let timelineHeight = 40; // Height of the timeline component
  
  // Shared state
  export let notes: Array<{
    id: string,
    start: number,
    duration: number,
    pitch: number,
    velocity: number,
    lyric?: string
  }> = [];
  
  // Settings
  export let tempo = 120;
  export let timeSignature = { numerator: 4, denominator: 4 };
  export let editMode = 'draw'; // 'draw', 'erase', 'select', etc.
  
  // Scroll positions
  let horizontalScroll = 0;
  let verticalScroll = 0;
  
  // References to DOM elements
  let containerElement: HTMLDivElement;
  
  // Sync scroll handlers
  function handleGridScroll(event: CustomEvent) {
    horizontalScroll = event.detail.horizontalScroll;
    verticalScroll = event.detail.verticalScroll;
  }
  
  onMount(() => {
    // Initialization code
  });
  
  onDestroy(() => {
    // Cleanup code
  });
</script>

<div 
  class="piano-roll-container" 
  bind:this={containerElement}
  style="width: {width}px; height: {height}px;"
>
  <Toolbar 
    {tempo}
    {timeSignature}
    {editMode}
    on:tempoChange={(e) => tempo = e.detail}
    on:timeSignatureChange={(e) => timeSignature = e.detail}
    on:editModeChange={(e) => editMode = e.detail}
  />
  
  <div class="piano-roll-main" style="height: {height - 40}px;">
    <KeyboardComponent 
      {keyboardWidth}
      height={height - 40 - timelineHeight}
      {verticalScroll}
    />
    
    <div class="grid-and-timeline">
      <TimeLineComponent
        width={width - keyboardWidth}
        {timelineHeight}
        {timeSignature}
        {horizontalScroll}
      />
      
      <GridComponent
        width={width - keyboardWidth}
        height={height - 40 - timelineHeight}
        {notes}
        {tempo}
        {timeSignature}
        {editMode}
        {horizontalScroll}
        {verticalScroll}
        on:scroll={handleGridScroll}
        on:noteChange
      />
    </div>
  </div>
</div>

<style>
  .piano-roll-container {
    display: flex;
    flex-direction: column;
    overflow: hidden;
    background-color: #2c2c2c;
    border-radius: 5px;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  }
  
  .piano-roll-main {
    display: flex;
    flex-direction: row;
    flex: 1;
  }
  
  .grid-and-timeline {
    display: flex;
    flex-direction: column;
    flex: 1;
  }
</style>
