<!--
  GridComponent for displaying and editing notes and lyrics.
  This component uses canvas to render the grid, notes, and lyrics.
-->
<script lang="ts">
  import { onMount, createEventDispatcher } from 'svelte';
  
  // Props
  export let width = 880;  // Width of the grid (total width - keyboard width)
  export let height = 520;  // Height of the grid
  export let notes: Array<{
    id: string,
    start: number,
    duration: number,
    pitch: number,
    velocity: number,
    lyric?: string
  }> = [];
  // Tempo is used to calculate timing and note positioning
  export let tempo = 120;  // BPM
  
  // Calculate pixels per second based on tempo
  $: pixelsPerSecond = (tempo / 60) * PIXELS_PER_BEAT;
  export let timeSignature = { numerator: 4, denominator: 4 };
  export let editMode = 'draw';  // Current edit mode
  export let horizontalScroll = 0;  // Horizontal scroll position
  export let verticalScroll = 0;  // Vertical scroll position
  
  // Constants
  const NOTE_HEIGHT = 20;  // Height of a note row (same as white key height)
  const GRID_COLOR = '#444444';
  const BEAT_COLOR = '#555555';
  const MEASURE_COLOR = '#666666';
  const NOTE_COLOR = '#2196F3';
  const NOTE_SELECTED_COLOR = '#03A9F4';
  const LYRIC_COLOR = '#FFFFFF';
  
  // Sizing and grid constants
  const TOTAL_NOTES = 128;  // Total MIDI notes
  const GRID_LINE_INTERVAL = NOTE_HEIGHT;  // Distance between horizontal grid lines
  const PIXELS_PER_BEAT = 80;  // How many pixels wide a beat is
  
  // State
  let canvas: HTMLCanvasElement;
  let ctx: CanvasRenderingContext2D | null = null;
  let isDragging = false;
  let isResizing = false;
  let selectedNotes: Set<string> = new Set();
  let dragStartX = 0;
  let dragStartY = 0;
  let lastMouseX = 0;
  let lastMouseY = 0;
  let draggedNoteId: string | null = null;
  let resizedNoteId: string | null = null;
  
  const dispatch = createEventDispatcher();
  
  // Calculate various dimensions and metrics
  $: totalGridHeight = TOTAL_NOTES * NOTE_HEIGHT;
  $: beatsPerMeasure = timeSignature.numerator;
  $: pixelsPerMeasure = beatsPerMeasure * PIXELS_PER_BEAT;
  
  // Calculate how many measures to show based on width
  $: totalMeasures = 32;  // Adjustable
  $: totalGridWidth = totalMeasures * pixelsPerMeasure;
  
  // Handle scrolling
  function handleScroll(event: WheelEvent) {
    event.preventDefault();
    
    // Vertical scrolling with mouse wheel
    if (event.deltaY !== 0) {
      const newVerticalScroll = Math.max(
        0, 
        Math.min(
          totalGridHeight - height, 
          verticalScroll + event.deltaY
        )
      );
      
      if (newVerticalScroll !== verticalScroll) {
        verticalScroll = newVerticalScroll;
        dispatch('scroll', { horizontalScroll, verticalScroll });
      }
    }
    
    // Horizontal scrolling with shift+wheel or trackpad
    if (event.deltaX !== 0 || event.shiftKey) {
      const deltaX = event.deltaX || event.deltaY;
      const newHorizontalScroll = Math.max(
        0, 
        Math.min(
          totalGridWidth - width, 
          horizontalScroll + deltaX
        )
      );
      
      if (newHorizontalScroll !== horizontalScroll) {
        horizontalScroll = newHorizontalScroll;
        dispatch('scroll', { horizontalScroll, verticalScroll });
      }
    }
    
    // Redraw with new scroll positions
    drawGrid();
  }
  
  // Mouse events for note manipulation
  function handleMouseDown(event: MouseEvent) {
    if (!canvas) return;
    
    const rect = canvas.getBoundingClientRect();
    const x = event.clientX - rect.left + horizontalScroll;
    const y = event.clientY - rect.top + verticalScroll;
    
    dragStartX = x;
    dragStartY = y;
    lastMouseX = x;
    lastMouseY = y;
    
    // Check if clicking on a note
    const clickedNote = findNoteAtPosition(x, y);
    
    if (editMode === 'draw' && !clickedNote) {
      // Create a new note
      const pitch = Math.floor(y / NOTE_HEIGHT);
      const time = snapToGrid(x);
      
      const newNote = {
        id: `note-${Date.now()}-${Math.random().toString(36).substr(2, 5)}`,
        start: time,
        duration: PIXELS_PER_BEAT / 2,  // Default to 1/8 note
        pitch: TOTAL_NOTES - 1 - pitch,
        velocity: 100,
        lyric: ''
      };
      
      notes = [...notes, newNote];
      selectedNotes = new Set([newNote.id]);
      draggedNoteId = newNote.id;
      isDragging = true;
      
      dispatch('noteChange', { notes });
    } 
    else if (editMode === 'erase' && clickedNote) {
      // Erase clicked note
      notes = notes.filter(note => note.id !== clickedNote.id);
      selectedNotes.delete(clickedNote.id);
      
      dispatch('noteChange', { notes });
    }
    else if (editMode === 'select') {
      if (clickedNote) {
        // Check if clicking near the end of the note (for resizing)
        const noteEndX = clickedNote.start + clickedNote.duration;
        if (Math.abs(x - noteEndX) < 10) {
          // Start resizing
          isResizing = true;
          resizedNoteId = clickedNote.id;
        } else {
          // Select and drag note
          if (!event.shiftKey) {
            // If not holding shift, clear previous selection
            if (!selectedNotes.has(clickedNote.id)) {
              selectedNotes = new Set([clickedNote.id]);
            }
          } else {
            // Add to selection with shift key
            selectedNotes.add(clickedNote.id);
          }
          
          isDragging = true;
          draggedNoteId = clickedNote.id;
        }
      } else {
        // Clicked empty space, clear selection unless shift is held
        if (!event.shiftKey) {
          selectedNotes = new Set();
        }
      }
    }
    
    // Redraw
    drawGrid();
  }
  
  function handleMouseMove(event: MouseEvent) {
    if (!canvas) return;
    
    const rect = canvas.getBoundingClientRect();
    const x = event.clientX - rect.left + horizontalScroll;
    const y = event.clientY - rect.top + verticalScroll;
    
    const deltaX = x - lastMouseX;
    const deltaY = y - lastMouseY;
    
    lastMouseX = x;
    lastMouseY = y;
    
    if (isDragging && draggedNoteId && editMode === 'select') {
      // Move selected notes
      notes = notes.map(note => {
        if (selectedNotes.has(note.id)) {
          const snappedDeltaX = (Math.round(deltaX / (PIXELS_PER_BEAT / 4)) * (PIXELS_PER_BEAT / 4));
          return {
            ...note,
            start: Math.max(0, note.start + snappedDeltaX),
            pitch: Math.max(0, Math.min(127, note.pitch - Math.round(deltaY / NOTE_HEIGHT)))
          };
        }
        return note;
      });
      
      dispatch('noteChange', { notes });
      drawGrid();
    } 
    else if (isResizing && resizedNoteId) {
      // Resize note
      notes = notes.map(note => {
        if (note.id === resizedNoteId) {
          const snappedDeltaX = (Math.round(deltaX / (PIXELS_PER_BEAT / 4)) * (PIXELS_PER_BEAT / 4));
          return {
            ...note,
            duration: Math.max(PIXELS_PER_BEAT / 8, note.duration + snappedDeltaX)
          };
        }
        return note;
      });
      
      dispatch('noteChange', { notes });
      drawGrid();
    }
  }
  
  function handleMouseUp() {
    isDragging = false;
    isResizing = false;
    draggedNoteId = null;
    resizedNoteId = null;
  }
  
  // Helper to find a note at a specific position
  function findNoteAtPosition(x: number, y: number) {
    const pitch = TOTAL_NOTES - 1 - Math.floor(y / NOTE_HEIGHT);
    
    return notes.find(note => {
      const noteY = (TOTAL_NOTES - 1 - note.pitch) * NOTE_HEIGHT;
      return (
        x >= note.start && 
        x <= note.start + note.duration &&
        y >= noteY && 
        y <= noteY + NOTE_HEIGHT
      );
    });
  }
  
  // Snap value to grid based on current beat division
  function snapToGrid(value: number) {
    const gridSize = PIXELS_PER_BEAT / 4; // 16th note grid
    return Math.round(value / gridSize) * gridSize;
  }
  
  // Convert beat position to pixel position
  function beatToPixel(beat: number) {
    return beat * PIXELS_PER_BEAT;
  }
  
  // Draw the grid with notes
  function drawGrid() {
    if (!ctx || !canvas) return;
    
    // Clear canvas
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    
    // Draw background
    ctx.fillStyle = '#2c2c2c';
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    
    // Calculate visible area
    const startX = horizontalScroll;
    const endX = horizontalScroll + width;
    const startY = verticalScroll;
    const endY = verticalScroll + height;
    
    // Draw vertical grid lines (beat and measure lines)
    const startMeasure = Math.floor(startX / pixelsPerMeasure);
    const endMeasure = Math.ceil(endX / pixelsPerMeasure);
    
    for (let measure = startMeasure; measure <= endMeasure; measure++) {
      const measureX = measure * pixelsPerMeasure - horizontalScroll;
      
      // Draw measure line
      ctx.strokeStyle = MEASURE_COLOR;
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(measureX, 0);
      ctx.lineTo(measureX, height);
      ctx.stroke();
      
      // Draw beat lines within measure
      for (let beat = 1; beat < beatsPerMeasure; beat++) {
        const beatX = measureX + beat * PIXELS_PER_BEAT;
        
        ctx.strokeStyle = BEAT_COLOR;
        ctx.lineWidth = 1;
        ctx.beginPath();
        ctx.moveTo(beatX, 0);
        ctx.lineTo(beatX, height);
        ctx.stroke();
      }
      
      // Draw subdivision lines (16th notes)
      for (let beat = 0; beat < beatsPerMeasure; beat++) {
        for (let tick = 1; tick < 4; tick++) {
          const tickX = measureX + beat * PIXELS_PER_BEAT + tick * (PIXELS_PER_BEAT / 4);
          
          ctx.strokeStyle = GRID_COLOR;
          ctx.lineWidth = 0.5;
          ctx.beginPath();
          ctx.moveTo(tickX, 0);
          ctx.lineTo(tickX, height);
          ctx.stroke();
        }
      }
    }
    
    // Draw horizontal grid lines
    const startRow = Math.floor(startY / GRID_LINE_INTERVAL);
    const endRow = Math.ceil(endY / GRID_LINE_INTERVAL);
    
    for (let row = startRow; row <= endRow; row++) {
      const rowY = row * GRID_LINE_INTERVAL - verticalScroll;
      
      // Draw row background for black keys (C#, D#, F#, G#, A#)
      const midiNote = TOTAL_NOTES - 1 - row;
      const noteIndex = midiNote % 12;
      
      if ([1, 3, 6, 8, 10].includes(noteIndex)) {
        ctx.fillStyle = 'rgba(0, 0, 0, 0.2)';
        ctx.fillRect(0, rowY, width, GRID_LINE_INTERVAL);
      }
      
      // Draw grid line
      ctx.strokeStyle = GRID_COLOR;
      ctx.lineWidth = 0.5;
      ctx.beginPath();
      ctx.moveTo(0, rowY);
      ctx.lineTo(width, rowY);
      ctx.stroke();
    }
    
    // Draw notes
    for (const note of notes) {
      const noteX = note.start - horizontalScroll;
      const noteY = (TOTAL_NOTES - 1 - note.pitch) * NOTE_HEIGHT - verticalScroll;
      
      // Skip notes outside of visible area
      if (
        noteX + note.duration < 0 || 
        noteX > width || 
        noteY + NOTE_HEIGHT < 0 || 
        noteY > height
      ) {
        continue;
      }
      
      // Draw note rectangle
      ctx.fillStyle = selectedNotes.has(note.id) ? NOTE_SELECTED_COLOR : NOTE_COLOR;
      ctx.fillRect(noteX, noteY, note.duration, NOTE_HEIGHT);
      
      // Draw border
      ctx.strokeStyle = '#1a1a1a';
      ctx.lineWidth = 1;
      ctx.strokeRect(noteX, noteY, note.duration, NOTE_HEIGHT);
      
      // Draw velocity indicator (brightness of note)
      const velocityHeight = (NOTE_HEIGHT - 4) * (note.velocity / 127);
      ctx.fillStyle = 'rgba(255, 255, 255, 0.2)';
      ctx.fillRect(noteX + 2, noteY + 2 + (NOTE_HEIGHT - 4 - velocityHeight), note.duration - 4, velocityHeight);
      
      // Draw lyric text if present and note is wide enough
      if (note.lyric && note.duration > 20) {
        ctx.fillStyle = LYRIC_COLOR;
        ctx.font = '10px Arial';
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        
        // Create text that fits within note width
        let text = note.lyric;
        const maxWidth = note.duration - 6;
        let textWidth = ctx.measureText(text).width;
        
        if (textWidth > maxWidth) {
          text = text.substring(0, Math.floor(text.length * (maxWidth / textWidth))) + '...';
        }
        
        ctx.fillText(text, noteX + note.duration / 2, noteY + NOTE_HEIGHT / 2);
      }
    }
  }
  
  // Set up the component
  onMount(() => {
    // Get canvas context
    ctx = canvas.getContext('2d');
    
    // Set up canvas size
    canvas.width = width;
    canvas.height = height;
    
    // Draw initial grid
    drawGrid();
  });
  
  // Update when props change
  $: {
    if (ctx && canvas) {
      canvas.width = width;
      canvas.height = height;
      drawGrid();
    }
  }
</script>

<canvas 
  bind:this={canvas} 
  width={width} 
  height={height}
  on:wheel={handleScroll}
  on:mousedown={handleMouseDown}
  on:mousemove={handleMouseMove}
  on:mouseup={handleMouseUp}
  on:mouseleave={handleMouseUp}
  class="grid-canvas"
></canvas>

<style>
  .grid-canvas {
    display: block;
    cursor: crosshair;
  }
</style>
