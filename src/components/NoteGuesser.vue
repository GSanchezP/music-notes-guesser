<template>
  <div class="note-guesser">
    <div class="octave-selector">
      <span class="octave-label">Octaves:</span>
      <label
        v-for="oct in availableOctaves"
        :key="oct"
        class="octave-option"
      >
        <input
          type="checkbox"
          :value="oct"
          v-model="selectedOctaves"
          :disabled="selectedOctaves.length <= 1 && selectedOctaves.includes(oct)"
        />
        {{ oct }}
      </label>
    </div>

    <div class="instrument-selector">
      <span class="instrument-label">Instrument:</span>
      <select v-model="selectedInstrument" class="instrument-select">
        <option
          v-for="inst in instruments"
          :key="inst.id"
          :value="inst.id"
        >
          {{ inst.name }}
        </option>
      </select>
    </div>

    <div class="controls">
      <button
        class="play-btn"
        @click="playNote"
        :disabled="isPlaying || selectedOctaves.length === 0"
      >
        {{ isPlaying ? '♪ Playing...' : '▶ Play Note' }}
      </button>
    </div>

    <div class="feedback" v-if="lastGuess !== null">
      <div
        class="feedback-message"
        :class="{ correct: wasCorrect, incorrect: !wasCorrect }"
      >
        {{ wasCorrect ? '✓ Correct!' : '✗ Wrong' }}
      </div>
      <p class="feedback-detail">
        The note was <strong>{{ correctNote }}{{ correctOctave }}</strong> ({{ NOTE_SPANISH[correctNote] }})
      </p>
    </div>

    <div class="note-buttons">
      <button
        v-for="note in notes"
        :key="note"
        class="note-btn"
        @click="guess(note)"
        :disabled="currentNote === null"
      >
        <span class="note-letter">{{ note }}</span>
        <span class="note-spanish">{{ NOTE_SPANISH[note] }}</span>
      </button>
    </div>

    <p class="hint" v-if="currentNote">Press a button to guess the note you heard</p>
    <p class="hint muted" v-else>Click "Play Note" to start</p>
  </div>
</template>

<script setup>
import { ref, onUnmounted } from 'vue'

const notes = ['C', 'D', 'E', 'F', 'G', 'A', 'B']
const NOTE_SPANISH = { C: 'Do', D: 'Re', E: 'Mi', F: 'Fa', G: 'Sol', A: 'La', B: 'Si' }
const availableOctaves = [2, 3, 4, 5]
const REPEAT_INTERVAL = 2000 // ms between each play

const instruments = [
  { id: 'sine', name: 'Pure tone' },
  { id: 'flute', name: 'Flute' },
  { id: 'piano', name: 'Piano' },
  { id: 'guitar', name: 'Guitar' },
]

const selectedOctaves = ref([4]) // Default: octave 4
const selectedInstrument = ref('sine')
const currentNote = ref(null)
const currentOctave = ref(null)
const lastGuess = ref(null)
const wasCorrect = ref(false)
const correctNote = ref('')
const correctOctave = ref('')
const isPlaying = ref(false)
let repeatIntervalId = null
let audioContext = null

function getAudioContext() {
  if (!audioContext) {
    audioContext = new (window.AudioContext || window.webkitAudioContext)()
  }
  return audioContext
}

// MIDI note to frequency: f = 440 * 2^((n-69)/12)
// Note offsets from C: C=0, D=2, E=4, F=5, G=7, A=9, B=11
const NOTE_OFFSETS = { C: 0, D: 2, E: 4, F: 5, G: 7, A: 9, B: 11 }

function getFrequency(note, octave) {
  const midi = 12 * (octave + 1) + NOTE_OFFSETS[note]
  return 440 * Math.pow(2, (midi - 69) / 12)
}

function playTone(frequency, duration = REPEAT_INTERVAL) {
  const ctx = getAudioContext()
  const now = ctx.currentTime
  const instrument = selectedInstrument.value

  if (instrument === 'sine') {
    const osc = ctx.createOscillator()
    const gain = ctx.createGain()
    osc.connect(gain)
    gain.connect(ctx.destination)
    osc.frequency.value = frequency
    osc.type = 'sine'
    gain.gain.setValueAtTime(0.3, now)
    gain.gain.exponentialRampToValueAtTime(0.001, now + duration / 1000)
    osc.start(now)
    osc.stop(now + duration / 1000)
    return
  }

  if (instrument === 'flute') {
    const osc1 = ctx.createOscillator()
    const osc2 = ctx.createOscillator()
    const osc2Gain = ctx.createGain()
    const masterGain = ctx.createGain()
    osc1.connect(masterGain)
    osc2.connect(osc2Gain)
    osc2Gain.connect(masterGain)
    masterGain.connect(ctx.destination)
    osc1.type = 'sine'
    osc2.type = 'sine'
    osc1.frequency.value = frequency
    osc2.frequency.value = frequency * 2
    osc2Gain.gain.setValueAtTime(0.08, now)
    masterGain.gain.setValueAtTime(0, now)
    masterGain.gain.linearRampToValueAtTime(0.25, now + 0.02)
    masterGain.gain.setValueAtTime(0.2, now + 0.05)
    masterGain.gain.exponentialRampToValueAtTime(0.001, now + duration / 1000)
    osc1.start(now)
    osc2.start(now)
    osc1.stop(now + duration / 1000)
    osc2.stop(now + duration / 1000)
    return
  }

  if (instrument === 'piano') {
    const harmonics = [
      { freq: 1, gain: 1 },
      { freq: 2, gain: 0.4 },
      { freq: 3, gain: 0.2 },
      { freq: 4, gain: 0.1 },
    ]
    const masterGain = ctx.createGain()
    masterGain.connect(ctx.destination)
    masterGain.gain.setValueAtTime(0, now)
    masterGain.gain.linearRampToValueAtTime(0.4, now + 0.005)
    masterGain.gain.exponentialRampToValueAtTime(0.001, now + Math.min(duration / 1000, 0.8))
    harmonics.forEach(({ freq, gain: hGain }) => {
      const osc = ctx.createOscillator()
      const oscGain = ctx.createGain()
      osc.connect(oscGain)
      oscGain.connect(masterGain)
      osc.type = 'sine'
      osc.frequency.value = frequency * freq
      oscGain.gain.setValueAtTime(hGain, now)
      osc.start(now)
      osc.stop(now + duration / 1000)
    })
    return
  }

  if (instrument === 'guitar') {
    const harmonics = [
      { freq: 1, gain: 1 },
      { freq: 2, gain: 0.35 },
      { freq: 2.5, gain: 0.15 },
      { freq: 3, gain: 0.08 },
    ]
    const masterGain = ctx.createGain()
    masterGain.connect(ctx.destination)
    masterGain.gain.setValueAtTime(0, now)
    masterGain.gain.linearRampToValueAtTime(0.35, now + 0.003)
    masterGain.gain.exponentialRampToValueAtTime(0.001, now + Math.min(duration / 1000, 0.6))
    harmonics.forEach(({ freq, gain: hGain }) => {
      const osc = ctx.createOscillator()
      const oscGain = ctx.createGain()
      osc.connect(oscGain)
      oscGain.connect(masterGain)
      osc.type = 'triangle'
      osc.frequency.value = frequency * freq
      oscGain.gain.setValueAtTime(hGain, now)
      osc.start(now)
      osc.stop(now + duration / 1000)
    })
    return
  }
}

function pickRandomNote() {
  const note = notes[Math.floor(Math.random() * notes.length)]
  const octaves = selectedOctaves.value.length > 0 ? selectedOctaves.value : availableOctaves
  const octave = octaves[Math.floor(Math.random() * octaves.length)]
  return { note, octave }
}

function stopRepeating() {
  if (repeatIntervalId) {
    clearInterval(repeatIntervalId)
    repeatIntervalId = null
  }
}

function startRepeatingNote(note, octave) {
  stopRepeating()
  const freq = getFrequency(note, octave)
  playTone(freq)
  repeatIntervalId = setInterval(() => playTone(freq), REPEAT_INTERVAL)
}

function playNote() {
  if (isPlaying.value) return
  const { note, octave } = pickRandomNote()
  currentNote.value = note
  currentOctave.value = octave
  lastGuess.value = null

  isPlaying.value = true
  startRepeatingNote(note, octave)
}

function guess(guessedNote) {
  if (currentNote.value === null) return

  stopRepeating()
  isPlaying.value = false

  wasCorrect.value = guessedNote === currentNote.value
  correctNote.value = currentNote.value
  correctOctave.value = currentOctave.value
  lastGuess.value = guessedNote

  // Reset and pick new note after a short delay
  currentNote.value = null
  currentOctave.value = null

  setTimeout(() => {
    const { note, octave } = pickRandomNote()
    currentNote.value = note
    currentOctave.value = octave
    lastGuess.value = null
    isPlaying.value = true
    startRepeatingNote(note, octave)
  }, 1500)
}

onUnmounted(() => {
  stopRepeating()
})
</script>

<style scoped>
.note-guesser {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1.25rem;
  padding: 1rem;
}

@media (min-width: 640px) {
  .note-guesser {
    gap: 2rem;
    padding: 2rem;
  }
}

.octave-selector {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  flex-wrap: wrap;
  justify-content: center;
}

@media (min-width: 640px) {
  .octave-selector {
    gap: 0.75rem;
  }
}

.octave-label {
  font-size: 0.875rem;
  color: #94a3b8;
  width: 100%;
  text-align: center;
}

@media (min-width: 640px) {
  .octave-label {
    width: auto;
    font-size: 0.9rem;
  }
}

.octave-option {
  display: flex;
  align-items: center;
  gap: 0.35rem;
  font-size: 1rem;
  font-weight: 600;
  color: #e2e8f0;
  cursor: pointer;
  padding: 0.5rem 0.875rem;
  min-height: 44px;
  background: #16213e;
  border: 2px solid #0f3460;
  border-radius: 8px;
  transition: all 0.15s;
  -webkit-tap-highlight-color: transparent;
  touch-action: manipulation;
}

.octave-option:has(input:checked) {
  background: #0f3460;
  border-color: #e94560;
}

.octave-option input {
  cursor: pointer;
  accent-color: #e94560;
  min-width: 20px;
  min-height: 20px;
}

.octave-option input:disabled {
  cursor: not-allowed;
  opacity: 0.7;
}

.instrument-selector {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  flex-wrap: wrap;
  justify-content: center;
}

.instrument-label {
  font-size: 0.875rem;
  color: #94a3b8;
}

.instrument-select {
  font-family: inherit;
  font-size: 1rem;
  font-weight: 600;
  padding: 0.5rem 0.875rem;
  min-height: 44px;
  background: #16213e;
  color: #e2e8f0;
  border: 2px solid #0f3460;
  border-radius: 8px;
  cursor: pointer;
  -webkit-tap-highlight-color: transparent;
  touch-action: manipulation;
}

.instrument-select:focus {
  outline: none;
  border-color: #e94560;
}

.controls {
  margin-bottom: 0.5rem;
  width: 100%;
}

.play-btn {
  width: 100%;
  padding: 1rem 1.5rem;
  min-height: 48px;
  font-size: 1.125rem;
  font-weight: 600;
  font-family: inherit;
  background: linear-gradient(135deg, #e94560 0%, #c73e54 100%);
  color: white;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  transition: transform 0.15s, box-shadow 0.15s;
  box-shadow: 0 4px 20px rgba(233, 69, 96, 0.4);
  -webkit-tap-highlight-color: transparent;
  touch-action: manipulation;
}

@media (min-width: 640px) {
  .play-btn {
    width: auto;
    padding: 1rem 2rem;
    font-size: 1.25rem;
  }
}

.play-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 6px 24px rgba(233, 69, 96, 0.5);
}

.play-btn:disabled {
  opacity: 0.8;
  cursor: not-allowed;
}

.feedback {
  text-align: center;
  min-height: 3.5rem;
}

.feedback-message {
  font-size: 1.375rem;
  font-weight: 700;
  margin-bottom: 0.5rem;
}

@media (min-width: 640px) {
  .feedback-message {
    font-size: 1.5rem;
  }
}

.feedback-message.correct {
  color: #4ade80;
}

.feedback-message.incorrect {
  color: #f87171;
}

.feedback-detail {
  color: #94a3b8;
  font-size: 0.9375rem;
  margin: 0;
}

@media (min-width: 640px) {
  .feedback-detail {
    font-size: 1rem;
  }
}

.note-buttons {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 0.5rem;
  width: 100%;
  max-width: 280px;
}

@media (min-width: 640px) {
  .note-buttons {
    display: flex;
    flex-wrap: wrap;
    gap: 0.75rem;
    max-width: none;
    justify-content: center;
  }
}

.note-btn {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 0.1rem;
  min-width: 0;
  min-height: 52px;
  aspect-ratio: 1;
  font-family: inherit;
  background: #16213e;
  color: #e2e8f0;
  border: 2px solid #0f3460;
  border-radius: 12px;
  cursor: pointer;
  transition: all 0.15s;
  -webkit-tap-highlight-color: transparent;
  touch-action: manipulation;
}

@media (min-width: 640px) {
  .note-btn {
    width: 4rem;
    height: 4.5rem;
    min-height: 4.5rem;
    aspect-ratio: auto;
  }
}

.note-btn .note-letter {
  font-size: 1.125rem;
  font-weight: 700;
}

@media (min-width: 640px) {
  .note-btn .note-letter {
    font-size: 1.25rem;
  }
}

.note-btn .note-spanish {
  font-size: 0.6875rem;
  font-weight: 500;
  opacity: 0.9;
}

@media (min-width: 640px) {
  .note-btn .note-spanish {
    font-size: 0.75rem;
  }
}

.note-btn:hover:not(:disabled) {
  background: #0f3460;
  border-color: #e94560;
  color: white;
  transform: scale(1.05);
}

@media (hover: none) {
  .note-btn:active:not(:disabled) {
    background: #0f3460;
    border-color: #e94560;
    color: white;
  }
}

.note-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.hint {
  color: #64748b;
  font-size: 0.875rem;
  margin: 0;
}

@media (min-width: 640px) {
  .hint {
    font-size: 0.9rem;
  }
}

.hint.muted {
  color: #475569;
}
</style>
