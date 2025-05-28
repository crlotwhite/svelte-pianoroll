# Svelte Piano Roll

Interactive piano roll editor built with Svelte, featuring real-time audio playback and waveform visualization.

## Features

- Interactive piano roll grid with note editing
- Real-time audio playback using Web Audio API
- Waveform visualization for audio feedback
- Precise timing using Facebook's Flicks system
- Playhead tracking during playback
- Play/pause/stop controls in the toolbar
- Debug information display for timing data

## Architecture

The piano roll is structured in layers:

1. **Top Layer**: Playhead and playback controls
2. **Middle Layer**: Notes and interactive elements
3. **Lower Layer**: Waveform visualization (semi-transparent)
4. **Bottom Layer**: Grid and reference lines

## Getting Started

Clone this repository and install dependencies:

```bash
git clone https://github.com/crlotwhite/svelte-pianoroll.git
cd svelte-pianoroll
npm install

# Start the development server
npm run dev -- --open
```

## Components

- **PianoRoll**: Main component that orchestrates all piano roll functionality
- **GridComponent**: Renders the background grid with time and pitch markings
- **WaveformComponent**: Displays audio visualization below the notes layer
- **PlayheadComponent**: Shows current playback position
- **DebugComponent**: Displays timing information for development
- **Toolbar**: Contains playback controls (play/pause/stop)

## Audio Engine

The audio engine uses the Web Audio API to synthesize and play notes in real-time, with precise timing managed using Facebook's Flicks system, which provides sub-millisecond accuracy.

## Building and Deployment

Build a production version:

```bash
npm run build
npm run preview
```

## License

MIT

## Acknowledgements

- [Svelte](https://svelte.dev/) - Frontend framework
- [Facebook's Flicks](https://github.com/OculusVR/Flicks) - Timing system
- [Web Audio API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API) - Audio processing
