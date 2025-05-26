<!--
  TimeLineComponent that displays measures and beats timeline.
-->
<script lang="ts">
  import { onMount } from 'svelte';
  
  // Props
  export let width = 880;  // Width of the timeline (same as grid width)
  export let timelineHeight = 40;  // Height of the timeline
  export let timeSignature = { numerator: 4, denominator: 4 };
  export let horizontalScroll = 0;  // Horizontal scroll position synced with GridComponent
  
  // Constants
  const PIXELS_PER_BEAT = 80;  // Should match GridComponent
  
  // Calculate appropriate subdivisions based on time signature denominator
  function getSubdivisionsFromTimeSignature(denominator: number): { count: number, pixelsPerSubdivision: number } {
    // The number of subdivisions per beat depends on the denominator
    switch (denominator) {
      case 2: // Half note gets the beat
        return { count: 2, pixelsPerSubdivision: PIXELS_PER_BEAT / 2 };
      case 4: // Quarter note gets the beat
        return { count: 4, pixelsPerSubdivision: PIXELS_PER_BEAT / 4 };
      case 8: // Eighth note gets the beat
        return { count: 3, pixelsPerSubdivision: PIXELS_PER_BEAT / 3 }; // Triplets
      case 16: // Sixteenth note gets the beat
        return { count: 4, pixelsPerSubdivision: PIXELS_PER_BEAT / 4 };
      default:
        return { count: 4, pixelsPerSubdivision: PIXELS_PER_BEAT / 4 }; // Default to 16th notes
    }
  }
  
  // Derived grid constants based on time signature
  $: subdivisions = getSubdivisionsFromTimeSignature(timeSignature.denominator);
  
  // DOM References
  let canvas: HTMLCanvasElement;
  let ctx: CanvasRenderingContext2D | null = null;
  
  // Calculate various dimensions
  $: beatsPerMeasure = timeSignature.numerator;
  $: pixelsPerMeasure = beatsPerMeasure * PIXELS_PER_BEAT;
  $: totalMeasures = 32;  // Should match GridComponent
  $: totalTimelineWidth = totalMeasures * pixelsPerMeasure;
  
  // Draw the timeline
  function drawTimeline() {
    if (!ctx || !canvas) return;
    
    // Clear canvas
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    
    // Draw background
    ctx.fillStyle = '#3a3a3a';
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    
    // Calculate visible area
    const startX = horizontalScroll;
    const endX = horizontalScroll + width;
    
    // Draw measure numbers and beat markers
    const startMeasure = Math.floor(startX / pixelsPerMeasure);
    const endMeasure = Math.ceil(endX / pixelsPerMeasure);
    
    for (let measure = startMeasure; measure <= endMeasure; measure++) {
      const measureX = measure * pixelsPerMeasure - horizontalScroll;
      
      // Draw measure number
      ctx.fillStyle = '#ffffff';
      ctx.font = '12px Arial';
      ctx.textAlign = 'center';
      ctx.textBaseline = 'top';
      ctx.fillText((measure + 1).toString(), measureX + pixelsPerMeasure / 2, 5);
      
      // Draw measure line
      ctx.strokeStyle = '#777777';
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(measureX, 0);
      ctx.lineTo(measureX, timelineHeight);
      ctx.stroke();
      
      // Draw beat lines
      for (let beat = 1; beat < beatsPerMeasure; beat++) {
        const beatX = measureX + beat * PIXELS_PER_BEAT;
        
        ctx.strokeStyle = '#666666';
        ctx.lineWidth = 1;
        ctx.beginPath();
        ctx.moveTo(beatX, timelineHeight / 2);
        ctx.lineTo(beatX, timelineHeight);
        ctx.stroke();
        
        // Draw beat number
        ctx.fillStyle = '#cccccc';
        ctx.font = '10px Arial';
        ctx.textAlign = 'center';
        ctx.textBaseline = 'bottom';
        ctx.fillText((beat + 1).toString(), beatX, timelineHeight - 5);
      }
      
      // Draw subdivision ticks based on time signature
      for (let beat = 0; beat < beatsPerMeasure; beat++) {
        for (let tick = 1; tick < subdivisions.count; tick++) {
          const tickX = measureX + beat * PIXELS_PER_BEAT + tick * subdivisions.pixelsPerSubdivision;
          
          ctx.strokeStyle = '#555555';
          ctx.lineWidth = 0.5;
          ctx.beginPath();
          ctx.moveTo(tickX, timelineHeight - 10);
          ctx.lineTo(tickX, timelineHeight);
          ctx.stroke();
        }
      }
    }
    
    // Draw playhead or other markers if needed
    // (Playback functionality could be added here)
  }
  
  // Set up the component
  onMount(() => {
    // Get canvas context
    ctx = canvas.getContext('2d');
    
    // Set up canvas size
    canvas.width = width;
    canvas.height = timelineHeight;
    
    // Draw initial timeline
    drawTimeline();
  });
  
  // Update when props change (width, height, etc.)
  $: {
    if (ctx && canvas) {
      canvas.width = width;
      canvas.height = timelineHeight;
      drawTimeline();
    }
  }
  
  // Redraw when time signature changes
  $: if (timeSignature && ctx && canvas) {
    // This will reactively update when timeSignature.numerator or denominator changes
    drawTimeline();
  }
  
  // Specifically redraw when horizontal scroll changes
  $: if (horizontalScroll !== undefined && ctx && canvas) {
    drawTimeline();
  }
</script>

<canvas 
  bind:this={canvas} 
  width={width} 
  height={timelineHeight}
  class="timeline-canvas"
></canvas>

<style>
  .timeline-canvas {
    display: block;
    background-color: #3a3a3a;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
  }
</style>
