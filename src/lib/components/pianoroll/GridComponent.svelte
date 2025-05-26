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
  
  // Derived grid constants based on time signature
  $: subdivisions = getSubdivisionsFromTimeSignature(timeSignature.denominator);
  
  // State
  let canvas: HTMLCanvasElement;
  let ctx: CanvasRenderingContext2D | null = null;
  let isDragging = false;
  let isResizing = false;
  let isCreatingNote = false;
  let selectedNotes: Set<string> = new Set();
  let dragStartX = 0;
  let dragStartY = 0;
  let lastMouseX = 0;
  let lastMouseY = 0;
  let draggedNoteId: string | null = null;
  let resizedNoteId: string | null = null;
  let creationStartTime = 0;
  let creationPitch = 0;
  
  // Lyric editing state
  let isEditingLyric = false;
  let editedNoteId: string | null = null;
  let lyricInputValue = '';
  let lyricInputPosition = { x: 0, y: 0, width: 0 };
  
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
      // Start note creation process
      const pitch = Math.floor(y / NOTE_HEIGHT);
      const time = snapToGrid(x);
      
      // Store the starting position and pitch for the new note
      creationStartTime = time;
      creationPitch = TOTAL_NOTES - 1 - pitch;
      
      // Create a new note with minimal duration initially
      const newNote = {
        id: `note-${Date.now()}-${Math.random().toString(36).substr(2, 5)}`,
        start: time,
        duration: PIXELS_PER_BEAT / 8,  // Start with a very small note
        pitch: creationPitch,
        velocity: 100,
        lyric: '라'  // Default lyric is '라'
      };
      
      // Add note to the collection
      notes = [...notes, newNote];
      
      // Set note as selected and being resized
      selectedNotes = new Set([newNote.id]);
      resizedNoteId = newNote.id;  // We're resizing, not dragging
      isCreatingNote = true;       // Flag that we're in note creation mode
      isResizing = true;          // Enable resizing mode
      
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
          let newDuration;
          
          if (isCreatingNote) {
            // For note creation, we want to resize from the start position to current mouse position
            // Calculate the width between the original start position and current mouse position
            const width = Math.max(PIXELS_PER_BEAT / 8, x - creationStartTime);
            // Snap the width to grid
            newDuration = Math.round(width / (PIXELS_PER_BEAT / 4)) * (PIXELS_PER_BEAT / 4);
            // Ensure minimum length
            newDuration = Math.max(PIXELS_PER_BEAT / 8, newDuration);
          } else {
            // For regular resizing, just add the delta to current duration
            const snappedDeltaX = (Math.round(deltaX / (PIXELS_PER_BEAT / 4)) * (PIXELS_PER_BEAT / 4));
            newDuration = Math.max(PIXELS_PER_BEAT / 8, note.duration + snappedDeltaX);
          }
          
          return {
            ...note,
            duration: newDuration
          };
        }
        return note;
      });
      
      dispatch('noteChange', { notes });
      drawGrid();
    }
  }
  
  function handleMouseUp() {
    // Check if we're finalizing note creation
    if (isCreatingNote) {
      // If the note is too small, remove it
      if (resizedNoteId) {
        const createdNote = notes.find(note => note.id === resizedNoteId);
        if (createdNote && createdNote.duration < PIXELS_PER_BEAT / 4) {
          // Remove notes that are too small (likely accidental clicks)
          notes = notes.filter(note => note.id !== resizedNoteId);
          dispatch('noteChange', { notes });
        }
      }
      
      // Reset creation state
      isCreatingNote = false;
    }
    
    // Reset interaction states
    isDragging = false;
    isResizing = false;
    draggedNoteId = null;
    resizedNoteId = null;
    
    // Redraw the grid
    drawGrid();
  }
  
  // Handle double-click to edit lyrics
  function handleDoubleClick(event: MouseEvent) {
    if (!canvas) return;
    
    const rect = canvas.getBoundingClientRect();
    const x = event.clientX - rect.left + horizontalScroll;
    const y = event.clientY - rect.top + verticalScroll;
    
    // Find the note that was double-clicked
    const clickedNote = findNoteAtPosition(x, y);
    
    if (clickedNote) {
      // Set up lyric editing state
      editedNoteId = clickedNote.id;
      lyricInputValue = clickedNote.lyric || '';
      
      // Calculate position for the input field
      const noteY = (TOTAL_NOTES - 1 - clickedNote.pitch) * NOTE_HEIGHT - verticalScroll;
      
      lyricInputPosition = {
        x: clickedNote.start - horizontalScroll,
        y: noteY,
        width: clickedNote.duration
      };
      
      isEditingLyric = true;
      
      // Set a timeout to focus the input element once it's rendered
      setTimeout(() => {
        const input = document.getElementById('lyric-input');
        if (input) {
          input.focus();
        }
      }, 10);
    }
  }
  
  // Save the edited lyric
  function saveLyric() {
    if (!editedNoteId) return;
    
    // Update the note with the new lyric
    notes = notes.map(note => {
      if (note.id === editedNoteId) {
        return {
          ...note,
          lyric: lyricInputValue
        };
      }
      return note;
    });
    
    // Reset editing state
    isEditingLyric = false;
    editedNoteId = null;
    
    // Notify parent of note changes
    dispatch('noteChange', { notes });
    
    // Redraw with updated lyrics
    drawGrid();
  }
  
  // Handle keydown in the lyric input
  function handleLyricInputKeydown(event: KeyboardEvent) {
    if (event.key === 'Enter') {
      saveLyric();
    } else if (event.key === 'Escape') {
      // Cancel editing
      isEditingLyric = false;
      editedNoteId = null;
    }
  }
  
  // Calculate appropriate subdivisions based on time signature denominator
  function getSubdivisionsFromTimeSignature(denominator: number): { count: number, pixelsPerSubdivision: number } {
    // The number of subdivisions per beat depends on the denominator
    // For example, in 4/4, we want 4 subdivisions (16th notes)
    // In 4/8, we want 3 subdivisions (triplets work well)
    // In 4/2, we want 2 subdivisions (8th notes)
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
      
      // Draw subdivision lines based on time signature
      for (let beat = 0; beat < beatsPerMeasure; beat++) {
        // Subdivisions based on time signature denominator
        for (let tick = 1; tick < subdivisions.count; tick++) {
          const tickX = measureX + beat * PIXELS_PER_BEAT + tick * subdivisions.pixelsPerSubdivision;
          
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
  
  // Redraw when time signature changes
  $: if (timeSignature && ctx && canvas) {
    // This will reactively update when timeSignature.numerator or denominator changes
    drawGrid();
  }
</script>

<div class="grid-container">
  <canvas 
    bind:this={canvas} 
    width={width} 
    height={height}
    on:wheel={handleScroll}
    on:mousedown={handleMouseDown}
    on:mousemove={handleMouseMove}
    on:mouseup={handleMouseUp}
    on:mouseleave={handleMouseUp}
    on:dblclick={handleDoubleClick}
    class="grid-canvas"
  ></canvas>
  
  {#if isEditingLyric}
    <div 
      class="lyric-input-container"
      style="
        left: {lyricInputPosition.x}px;
        top: {lyricInputPosition.y}px;
        width: {lyricInputPosition.width}px;
      "
    >
      <input 
        id="lyric-input"
        type="text" 
        bind:value={lyricInputValue}
        on:keydown={handleLyricInputKeydown}
        on:blur={saveLyric}
        class="lyric-input"
        aria-label="노트 가사 편집"
      />
    </div>
  {/if}
</div>

<style>
  .grid-container {
    position: relative;
    height: 100%;
  }
  
  .grid-canvas {
    display: block;
    cursor: crosshair;
  }
  
  .lyric-input-container {
    position: absolute;
    z-index: 10;
  }
  
  .lyric-input {
    width: 100%;
    height: 18px;
    background-color: #fff;
    border: 1px solid #1976D2;
    border-radius: 2px;
    font-size: 10px;
    padding: 0 4px;
    color: #333;
    box-sizing: border-box;
  }
  
  .lyric-input:focus {
    outline: none;
    box-shadow: 0 0 0 2px rgba(33, 150, 243, 0.4);
  }
</style>
