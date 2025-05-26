<!--
  Toolbar component for controlling tempo, time signature, and edit mode.
-->
<script lang="ts">
  import { createEventDispatcher } from 'svelte';
  
  export let tempo = 120;
  export let timeSignature = { numerator: 4, denominator: 4 };
  export let editMode = 'draw'; // 'draw', 'erase', 'select', etc.
  
  const dispatch = createEventDispatcher();
  
  // Edit mode options
  const editModes = [
    { id: 'draw', label: 'Draw', icon: '✏️' },
    { id: 'erase', label: 'Erase', icon: '🧹' },
    { id: 'select', label: 'Select', icon: '👆' },
  ];
  
  // Time signature numerator and denominator options
  const numeratorOptions = [2, 3, 4, 5, 6, 7, 9, 12];
  const denominatorOptions = [2, 4, 8, 16];
  
  function updateTempo(event: Event) {
    const newTempo = parseInt((event.target as HTMLInputElement).value);
    dispatch('tempoChange', newTempo);
  }
  
  function updateTimeSignature() {
    dispatch('timeSignatureChange', timeSignature);
  }
  
  function setEditMode(mode: string) {
    dispatch('editModeChange', mode);
  }
</script>

<div class="toolbar">
  <div class="section time-section">
    <div class="tempo-control">
      <label for="tempo-input">Tempo</label>
      <div class="tempo-input-group">
        <input 
          type="number" 
          id="tempo-input" 
          min="20" 
          max="300" 
          bind:value={tempo} 
          on:change={updateTempo}
        />
        <span class="unit">BPM</span>
      </div>
    </div>
    
    <div class="time-signature-control">
      <label id="time-signature-label">Signature</label>
      <div class="time-signature-input-group" aria-labelledby="time-signature-label">
        <select 
          id="time-sig-numerator"
          bind:value={timeSignature.numerator} 
          on:change={updateTimeSignature}
          aria-label="Time Signature Numerator"
        >
          {#each numeratorOptions as num}
            <option value={num}>{num}</option>
          {/each}
        </select>
        <span class="divider" aria-hidden="true">/</span>
        <select 
          id="time-sig-denominator"
          bind:value={timeSignature.denominator} 
          on:change={updateTimeSignature}
          aria-label="Time Signature Denominator"
        >
          {#each denominatorOptions as denom}
            <option value={denom}>{denom}</option>
          {/each}
        </select>
      </div>
    </div>
  </div>
  
  <div class="section edit-mode-section">
    <div id="edit-mode-label" class="edit-mode-label">Mode</div>
    <div class="edit-mode-buttons" aria-labelledby="edit-mode-label">
      {#each editModes as mode}
        <button 
          class="edit-mode-button {editMode === mode.id ? 'active' : ''}" 
          on:click={() => setEditMode(mode.id)}
          title={mode.label}
        >
          <span class="icon">{mode.icon}</span>
          <span class="label">{mode.label}</span>
        </button>
      {/each}
    </div>
  </div>
</div>

<style>
  .toolbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 8px 12px;
    height: 40px;
    background-color: #3a3a3a;
    border-bottom: 1px solid #555;
  }
  
  .section {
    display: flex;
    align-items: center;
  }
  
  label {
    font-size: 12px;
    margin-right: 8px;
    color: #ccc;
  }
  
  .tempo-control, .time-signature-control {
    display: flex;
    align-items: center;
    margin-right: 16px;
  }
  
  .tempo-input-group, .time-signature-input-group {
    display: flex;
    align-items: center;
  }
  
  input, select {
    background-color: #2c2c2c;
    border: 1px solid #555;
    border-radius: 3px;
    color: #fff;
    font-size: 12px;
    padding: 4px 6px;
    width: 50px;
  }
  
  .unit, .divider {
    color: #ccc;
    font-size: 12px;
    margin: 0 4px;
  }
  
  .edit-mode-section {
    display: flex;
    align-items: center;
  }
  
  .edit-mode-label {
    font-size: 12px;
    color: #ccc;
    margin-right: 8px;
  }
  
  .edit-mode-buttons {
    display: flex;
    gap: 4px;
  }
  
  .edit-mode-button {
    display: flex;
    align-items: center;
    background-color: #2c2c2c;
    border: 1px solid #555;
    border-radius: 3px;
    color: #ccc;
    cursor: pointer;
    font-size: 12px;
    padding: 4px 8px;
  }
  
  .edit-mode-button.active {
    background-color: #4a4a4a;
    color: #fff;
    border-color: #777;
  }
  
  .icon {
    margin-right: 4px;
  }
  
  @media (max-width: 768px) {
    .label {
      display: none;
    }
    
    .icon {
      margin-right: 0;
    }
  }
</style>
