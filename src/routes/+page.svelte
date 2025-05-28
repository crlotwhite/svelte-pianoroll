<script lang="ts">
  import PianoRoll from '../lib/components/pianoroll/PianoRoll.svelte';
  
  // Sample notes data
  const sampleNotes = [
    {
      id: 'note-1',
      start: 80, // 1st beat of measure 1
      duration: 80, // Quarter note
      pitch: 60, // Middle C
      velocity: 100,
      lyric: '라'
    },
    {
      id: 'note-2',
      start: 160, // 1st beat of measure 2
      duration: 160, // Half note
      pitch: 64, // E
      velocity: 90,
      lyric: '라'
    },
    {
      id: 'note-3',
      start: 320, // 1st beat of measure 3
      duration: 80, // Quarter note
      pitch: 67, // G
      velocity: 95,
      lyric: '라'
    }
  ];
</script>

<main>
  <h1>Svelte Piano Roll for Gradio</h1>
  <p>A piano roll component built with Svelte 4.x for use in Gradio applications.</p>
  
  <div class="piano-roll-wrapper">
    <PianoRoll width={1000} height={600} notes={sampleNotes} />
  </div>
  
  <div class="instructions">
    <h2>사용 방법:</h2>
    <ul>
      <li>툴바에서 템포, 박자, 편집 모드를 설정할 수 있습니다.</li>
      <li>키보드를 클릭하여 음을 미리 들을 수 있습니다.</li>
      <li>그리드에서 노트를 추가하고 편집할 수 있습니다.</li>
      <li>타임라인은 마디와 박자를 표시합니다.</li>
    </ul>
  </div>
</main>

<style>
  main {
    max-width: 1200px;
    margin: 0 auto;
    padding: 2rem;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  }
  
  h1 {
    color: #333;
    font-size: 2rem;
    margin-bottom: 1rem;
  }
  
  p {
    color: #666;
    margin-bottom: 2rem;
  }
  
  .piano-roll-wrapper {
    margin-bottom: 2rem;
    border-radius: 8px;
    overflow: hidden;
    box-shadow: 0 8px 24px rgba(0, 0, 0, 0.15);
  }
  
  .instructions {
    background-color: #f5f5f5;
    padding: 1.5rem;
    border-radius: 8px;
  }
  
  h2 {
    font-size: 1.25rem;
    margin-bottom: 1rem;
    color: #333;
  }
  
  ul {
    list-style-type: disc;
    padding-left: 1.5rem;
    color: #555;
  }
  
  li {
    margin-bottom: 0.5rem;
  }
</style>
