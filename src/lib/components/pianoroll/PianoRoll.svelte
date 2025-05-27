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
  export let editMode = 'select'; // 'select', 'draw', 'erase', etc.
  export let snapSetting = '1/4'; // Default snap setting: 1/4
  
  // Zoom level (pixels per beat)
  let pixelsPerBeat = 80; // Default zoom level
  const MIN_PIXELS_PER_BEAT = 40; // Minimum zoom level
  const MAX_PIXELS_PER_BEAT = 200; // Maximum zoom level
  const ZOOM_STEP = 20; // Zoom step size (must be integer to avoid coordinate calculation errors)
  
  // Zoom in function
  function zoomIn() {
    if (pixelsPerBeat < MAX_PIXELS_PER_BEAT) {
      pixelsPerBeat += ZOOM_STEP;
    }
  }
  
  // Zoom out function
  function zoomOut() {
    if (pixelsPerBeat > MIN_PIXELS_PER_BEAT) {
      pixelsPerBeat -= ZOOM_STEP;
    }
  }
  
  // Scroll positions
  let horizontalScroll = 0;
  let verticalScroll = 0;
  
  // References to DOM elements
  let containerElement: HTMLDivElement;
  
  // Sync scroll handlers
  function handleGridScroll(event: CustomEvent) {
    horizontalScroll = event.detail.horizontalScroll;
    verticalScroll = event.detail.verticalScroll;
    // The scroll values are now reactively bound to the other components
    // and will trigger updates when they change
  }
  
  // Settings handlers
  function handleTimeSignatureChange(event: CustomEvent) {
    timeSignature = event.detail;
  }
  
  function handleEditModeChange(event: CustomEvent) {
    editMode = event.detail;
  }
  
  function handleSnapChange(event: CustomEvent) {
    snapSetting = event.detail;
  }
  
  // Handle zoom changes from toolbar
  function handleZoomChange(event: CustomEvent) {
    const { action } = event.detail;
    if (action === 'zoom-in') {
      zoomIn();
    } else if (action === 'zoom-out') {
      zoomOut();
    }
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
    {snapSetting}
    on:tempoChange={(e) => tempo = e.detail}
    on:timeSignatureChange={handleTimeSignatureChange}
    on:editModeChange={handleEditModeChange}
    on:snapChange={handleSnapChange}
    on:zoomChange={handleZoomChange}
  />
  
  <div class="piano-roll-main" style="height: {height - 40}px;">
    <!-- Timeline positioned at the top -->
    <div class="timeline-container" style="margin-left: {keyboardWidth}px;">
      <TimeLineComponent
        width={width - keyboardWidth}
        {timelineHeight}
        {timeSignature}
        {snapSetting}
        {horizontalScroll}
        {pixelsPerBeat}
        on:zoomChange={handleZoomChange}
      />
    </div>
    
    <!-- Main content area with keyboard and grid aligned -->
    <div class="content-container">
      <KeyboardComponent 
        {keyboardWidth}
        height={height - 40 - timelineHeight}
        {verticalScroll}
      />
      
      <GridComponent
        width={width - keyboardWidth}
        height={height - 40 - timelineHeight}
        {notes}
        {tempo}
        {timeSignature}
        {editMode}
        {snapSetting}
        {horizontalScroll}
        {verticalScroll}
        {pixelsPerBeat}
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
    flex-direction: column;
    flex: 1;
  }
  
  .timeline-container {
    display: flex;
    height: var(--timeline-height, 40px);
  }
  
  .content-container {
    display: flex;
    flex-direction: row;
    flex: 1;
  }
</style>
