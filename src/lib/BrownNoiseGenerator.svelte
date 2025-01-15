<script lang="ts">
  import { onMount, onDestroy } from 'svelte'

  let audioContext: AudioContext | null = null
  let brownNoiseNode: AudioBufferSourceNode | null = null
  let gainNode: GainNode | null = null
  let analyserNode: AnalyserNode | null = null
  let leftOscillator: OscillatorNode | null = null
  let rightOscillator: OscillatorNode | null = null
  let leftGain: GainNode | null = null
  let rightGain: GainNode | null = null
  let stereoPanner: StereoPannerNode | null = null
  let isBrownNoisePlaying = false
  let isBinauralPlaying = false
  let volume = 0.5
  let baseFrequency = 200 // Base carrier frequency
  let beatFrequency = 0.5 // Default to Delta wave
  let showVolumeControl = false
  let canvas: HTMLCanvasElement | null = null
  let animationFrameId: number | null = null

  // Brainwave frequency presets
  const brainwavePresets = [
    {
      name: 'Brown Noise',
      frequency: null,
      description: 'Ambient focus, relaxation',
    },
    { name: 'Delta', frequency: 0.5, description: 'Deep sleep, healing' },
    { name: 'Theta', frequency: 4, description: 'Meditation, creativity' },
    { name: 'Alpha', frequency: 8, description: 'Relaxation, focus' },
    { name: 'Beta', frequency: 13, description: 'Active thinking, alertness' },
    { name: 'Gamma', frequency: 30, description: 'Peak performance' },
  ]

  function createBrownNoise(duration: number, sampleRate: number): AudioBuffer {
    const bufferSize = duration * sampleRate
    const audioBuffer = new AudioBuffer({
      length: bufferSize,
      numberOfChannels: 1,
      sampleRate: sampleRate,
    })

    const data = audioBuffer.getChannelData(0)
    let lastOut = 0

    for (let i = 0; i < bufferSize; i++) {
      const white = Math.random() * 2 - 1
      data[i] = (lastOut + 0.02 * white) / 1.02
      lastOut = data[i]
      data[i] *= 3.5
    }

    return audioBuffer
  }

  function setupAnalyser() {
    if (!audioContext) return
    analyserNode = audioContext.createAnalyser()
    analyserNode.fftSize = 2048
    if (gainNode) {
      gainNode.connect(analyserNode)
      analyserNode.connect(audioContext.destination)
    }
    // Start animation when audio is playing
    if ((isBrownNoisePlaying || isBinauralPlaying) && !animationFrameId) {
      drawWaveform()
    }
  }

  function drawWaveform() {
    if (!analyserNode || !canvas) return
    const ctx = canvas.getContext('2d')
    if (!ctx) return

    const bufferLength = analyserNode.frequencyBinCount
    const dataArray = new Uint8Array(bufferLength)
    analyserNode.getByteTimeDomainData(dataArray)

    // Clear canvas and set style
    ctx.clearRect(0, 0, canvas.width, canvas.height)
    ctx.fillStyle = 'rgba(42, 42, 42, 0.95)'
    ctx.fillRect(0, 0, canvas.width, canvas.height)

    // Draw waveform
    ctx.lineWidth = 2
    ctx.strokeStyle = '#ff3e00'
    ctx.beginPath()

    const sliceWidth = canvas.width / bufferLength
    let x = 0

    for (let i = 0; i < bufferLength; i++) {
      const v = dataArray[i] / 128.0
      const y = (v * canvas.height) / 2

      if (i === 0) {
        ctx.moveTo(x, y)
      } else {
        ctx.lineTo(x, y)
      }

      x += sliceWidth
    }

    ctx.lineTo(canvas.width, canvas.height / 2)
    ctx.stroke()

    animationFrameId = requestAnimationFrame(drawWaveform)
  }

  function setupBinauralBeats() {
    if (!audioContext) {
      audioContext = new AudioContext()
      gainNode = audioContext.createGain()
      setupAnalyser()
      gainNode.gain.value = volume
    }

    // Create oscillators for left and right ears
    leftOscillator = audioContext.createOscillator()
    rightOscillator = audioContext.createOscillator()

    leftGain = audioContext.createGain()
    rightGain = audioContext.createGain()

    // Set frequencies for binaural beat effect
    leftOscillator.frequency.value = baseFrequency
    rightOscillator.frequency.value = baseFrequency + beatFrequency

    // Create stereo separation
    const merger = audioContext.createChannelMerger(2)

    // Connect left oscillator to left channel
    leftOscillator.connect(leftGain)
    leftGain.connect(merger, 0, 0)

    // Connect right oscillator to right channel
    rightOscillator.connect(rightGain)
    rightGain.connect(merger, 0, 1)

    // Connect merger to main gain node
    if (gainNode) {
      merger.connect(gainNode)
    }

    // Set initial volume
    leftGain.gain.value = 0.1
    rightGain.gain.value = 0.1

    // Start oscillators
    leftOscillator.start()
    rightOscillator.start()
  }

  function updateBinauralBeats() {
    if (leftOscillator && rightOscillator) {
      leftOscillator.frequency.value = baseFrequency
      rightOscillator.frequency.value = baseFrequency + beatFrequency
    }
  }

  function setBrainwavePreset(frequency: number | null) {
    if (frequency === null) {
      // Handle Brown Noise
      toggleBrownNoise()
      if (isBinauralPlaying) {
        toggleBinauralBeats()
      }
    } else {
      beatFrequency = frequency
      if (isBrownNoisePlaying) {
        toggleBrownNoise()
      }
      if (!isBinauralPlaying) {
        toggleBinauralBeats()
      } else {
        updateBinauralBeats()
      }
    }

    // Start or stop animation based on playback state
    if ((isBrownNoisePlaying || isBinauralPlaying) && !animationFrameId) {
      drawWaveform()
    } else if (!isBrownNoisePlaying && !isBinauralPlaying && animationFrameId) {
      cancelAnimationFrame(animationFrameId)
      animationFrameId = null
    }
  }

  function toggleBrownNoise() {
    if (!audioContext) {
      audioContext = new AudioContext()
      gainNode = audioContext.createGain()
      setupAnalyser()
      gainNode.gain.value = volume
    }

    if (!isBrownNoisePlaying) {
      brownNoiseNode = audioContext.createBufferSource()
      brownNoiseNode.buffer = createBrownNoise(1, audioContext.sampleRate)
      brownNoiseNode.loop = true
      if (gainNode) {
        gainNode.gain.value = volume * 0.1
        brownNoiseNode.connect(gainNode)
      }
      brownNoiseNode.start()
      isBrownNoisePlaying = true
    } else {
      brownNoiseNode?.stop()
      brownNoiseNode?.disconnect()
      brownNoiseNode = null
      isBrownNoisePlaying = false
    }
  }

  function toggleBinauralBeats() {
    if (!isBinauralPlaying) {
      setupBinauralBeats()
      isBinauralPlaying = true
    } else {
      leftOscillator?.stop()
      rightOscillator?.stop()
      leftOscillator?.disconnect()
      rightOscillator?.disconnect()
      leftOscillator = null
      rightOscillator = null
      isBinauralPlaying = false
    }
  }

  function updateVolume() {
    if (gainNode) {
      if (isBrownNoisePlaying) {
        gainNode.gain.value = volume * 0.1
      } else {
        gainNode.gain.value = volume
      }
    }
  }

  onMount(() => {
    const resizeCanvas = () => {
      if (!canvas) return
      canvas.width = window.innerWidth
      canvas.height = 100
    }
    window.addEventListener('resize', resizeCanvas)
    resizeCanvas()

    return () => {
      window.removeEventListener('resize', resizeCanvas)
    }
  })

  onDestroy(() => {
    if (animationFrameId) {
      cancelAnimationFrame(animationFrameId)
      animationFrameId = null
    }
    if (brownNoiseNode) {
      brownNoiseNode.stop()
      brownNoiseNode.disconnect()
    }
    if (leftOscillator) {
      leftOscillator.stop()
      leftOscillator.disconnect()
    }
    if (rightOscillator) {
      rightOscillator.stop()
      rightOscillator.disconnect()
    }
    if (audioContext) {
      audioContext.close()
    }
  })
</script>

<div class="waveform-container">
  <canvas bind:this={canvas} />
</div>

<div class="noise-container">
  <div class="dock">
    <div
      class="volume-control"
      on:mouseenter={() => (showVolumeControl = true)}
      on:mouseleave={() => (showVolumeControl = false)}
    >
      <button class="dock-button volume-button">
        <span class="icon">🔊</span>
        <span class="label">Volume</span>
      </button>
      {#if showVolumeControl}
        <div class="volume-slider-tooltip">
          <input
            type="range"
            id="volume"
            min="0"
            max="1"
            step="0.01"
            bind:value={volume}
            on:input={updateVolume}
          />
        </div>
      {/if}
    </div>

    <div class="preset-buttons">
      {#each brainwavePresets as preset}
        <button
          class="dock-button preset-button"
          class:active={preset.frequency === null
            ? isBrownNoisePlaying
            : beatFrequency === preset.frequency && isBinauralPlaying}
          on:click={() => setBrainwavePreset(preset.frequency)}
        >
          <span class="icon">
            {#if preset.frequency === null}
              🌊
            {:else if preset.frequency <= 0.5}
              😴
            {:else if preset.frequency <= 4}
              🧘
            {:else if preset.frequency <= 8}
              😌
            {:else if preset.frequency <= 13}
              🧠
            {:else}
              ⚡
            {/if}
          </span>
          <span class="label">{preset.name}</span>
        </button>
      {/each}
    </div>
  </div>
</div>

<style>
  .waveform-container {
    position: fixed;
    bottom: 80px;
    left: 0;
    right: 0;
    height: 100px;
    pointer-events: none;
  }

  canvas {
    width: 100%;
    height: 100%;
  }

  .noise-container {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    padding: 1rem;
    background: rgba(42, 42, 42, 0.95);
    backdrop-filter: blur(10px);
    box-shadow: 0 -4px 6px rgba(0, 0, 0, 0.1);
  }

  .dock {
    display: flex;
    justify-content: center;
    align-items: center;
    gap: 0.5rem;
    max-width: 1200px;
    margin: 0 auto;
  }

  .preset-buttons {
    display: flex;
    gap: 0.5rem;
  }

  .dock-button {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0.25rem;
    padding: 0.5rem;
    min-width: 80px;
    background-color: #3a3a3a;
    border: 1px solid #4a4a4a;
    border-radius: 12px;
    color: white;
    cursor: pointer;
    transition: all 0.2s ease;
  }

  .dock-button:hover {
    background-color: #4a4a4a;
    transform: translateY(-5px);
  }

  .dock-button.active {
    background-color: #5a5a5a;
    border-color: #ff3e00;
  }

  .icon {
    font-size: 1.5em;
  }

  .label {
    font-size: 0.8em;
    text-align: center;
  }

  .volume-control {
    position: relative;
  }

  .volume-button {
    background-color: #2a2a2a;
  }

  .volume-slider-tooltip {
    position: absolute;
    bottom: 100%;
    left: 50%;
    transform: translateX(-50%);
    background: #2a2a2a;
    padding: 1rem;
    border-radius: 8px;
    margin-bottom: 1rem;
    width: 200px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  }

  .volume-slider-tooltip::after {
    content: '';
    position: absolute;
    bottom: -10px;
    left: 50%;
    transform: translateX(-50%);
    border-left: 10px solid transparent;
    border-right: 10px solid transparent;
    border-top: 10px solid #2a2a2a;
  }

  input[type='range'] {
    width: 100%;
    cursor: pointer;
    -webkit-appearance: none;
    background: #4a4a4a;
    height: 4px;
    border-radius: 2px;
  }

  input[type='range']::-webkit-slider-thumb {
    -webkit-appearance: none;
    width: 16px;
    height: 16px;
    background: #ff3e00;
    border-radius: 50%;
    cursor: pointer;
  }

  input[type='range']::-moz-range-thumb {
    width: 16px;
    height: 16px;
    background: #ff3e00;
    border-radius: 50%;
    cursor: pointer;
    border: none;
  }
</style>
