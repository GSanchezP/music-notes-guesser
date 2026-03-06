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

const selectedOctaves = ref([3, 4]) // Default: octaves 3 and 4
const currentNote = ref(null)
const currentOctave = ref(null)
const lastGuess = ref(null)
const wasCorrect = ref(false)
const correctNote = ref('')
const correctOctave = ref('')
const isPlaying = ref(false)
let repeatIntervalId = null

// MIDI note to frequency: f = 440 * 2^((n-69)/12)
// Note offsets from C: C=0, D=2, E=4, F=5, G=7, A=9, B=11
const NOTE_OFFSETS = { C: 0, D: 2, E: 4, F: 5, G: 7, A: 9, B: 11 }

function getFrequency(note, octave) {
  const midi = 12 * (octave + 1) + NOTE_OFFSETS[note]
  return 440 * Math.pow(2, (midi - 69) / 12)
}

function playTone(frequency, duration = REPEAT_INTERVAL) {
  const audioContext = new (window.AudioContext || window.webkitAudioContext)()
  const oscillator = audioContext.createOscillator()
  const gainNode = audioContext.createGain()

  oscillator.connect(gainNode)
  gainNode.connect(audioContext.destination)

  oscillator.frequency.value = frequency
  oscillator.type = 'sine'
  gainNode.gain.setValueAtTime(0.3, audioContext.currentTime)
  gainNode.gain.exponentialRampToValueAtTime(0.001, audioContext.currentTime + duration / 1000)

  oscillator.start(audioContext.currentTime)
  oscillator.stop(audioContext.currentTime + duration / 1000)
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
  gap: 2rem;
  padding: 2rem;
}

.octave-selector {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  flex-wrap: wrap;
  justify-content: center;
}

.octave-label {
  font-size: 0.9rem;
  color: #94a3b8;
}

.octave-option {
  display: flex;
  align-items: center;
  gap: 0.35rem;
  font-size: 1rem;
  font-weight: 600;
  color: #e2e8f0;
  cursor: pointer;
  padding: 0.4rem 0.75rem;
  background: #16213e;
  border: 2px solid #0f3460;
  border-radius: 8px;
  transition: all 0.15s;
}

.octave-option:has(input:checked) {
  background: #0f3460;
  border-color: #e94560;
}

.octave-option input {
  cursor: pointer;
  accent-color: #e94560;
}

.octave-option input:disabled {
  cursor: not-allowed;
  opacity: 0.7;
}

.controls {
  margin-bottom: 0.5rem;
}

.play-btn {
  padding: 1rem 2rem;
  font-size: 1.25rem;
  font-weight: 600;
  font-family: inherit;
  background: linear-gradient(135deg, #e94560 0%, #c73e54 100%);
  color: white;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  transition: transform 0.15s, box-shadow 0.15s;
  box-shadow: 0 4px 20px rgba(233, 69, 96, 0.4);
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
  min-height: 4rem;
}

.feedback-message {
  font-size: 1.5rem;
  font-weight: 700;
  margin-bottom: 0.5rem;
}

.feedback-message.correct {
  color: #4ade80;
}

.feedback-message.incorrect {
  color: #f87171;
}

.feedback-detail {
  color: #94a3b8;
  font-size: 1rem;
  margin: 0;
}

.note-buttons {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
  justify-content: center;
}

.note-btn {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 0.15rem;
  width: 4rem;
  height: 4.5rem;
  font-family: inherit;
  background: #16213e;
  color: #e2e8f0;
  border: 2px solid #0f3460;
  border-radius: 12px;
  cursor: pointer;
  transition: all 0.15s;
}

.note-btn .note-letter {
  font-size: 1.25rem;
  font-weight: 700;
}

.note-btn .note-spanish {
  font-size: 0.75rem;
  font-weight: 500;
  opacity: 0.9;
}

.note-btn:hover:not(:disabled) {
  background: #0f3460;
  border-color: #e94560;
  color: white;
  transform: scale(1.05);
}

.note-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.hint {
  color: #64748b;
  font-size: 0.9rem;
  margin: 0;
}

.hint.muted {
  color: #475569;
}
</style>
