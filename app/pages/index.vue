<template>
  <div>
    <div>
      <input type="file" accept="audio/midi" @change="onMidiFileChanged">
      <button @click="play">play</button>
      <button @click="picoAudio.pause">pause</button>
      <button @click="picoAudio.initStatus">init</button>
    </div>
    <div style="position: relative">
      <svg
        style="    isolation: isolate"
        :width="pianorollSize.width"
        :height="pianorollSize.height"
        :viewBox="`${pianorollX} ${pianorollY} ${pianorollSize.width} ${pianorollSize.height}`"
        xmlns="http://www.w3.org/2000/svg"
      >
        <g style="isolation:isolate" ref="pianoroll"></g>
      </svg>
      <div :style="{position: 'absolute', width: '1px', height: `${pianorollSize.height}px`, top: '0', left: `${pianorollSize.width / 2}px`, backgroundColor: '#888'}"></div>
    </div>
  </div>
</template>

<script setup lang="ts">
  import PicoAudio from 'picoaudio'
  import { ref, onMounted } from 'vue'

  const ChannelColor = [
    "#f66", "#fa6", "#f6f", "#6f6",
    "#6af", "#66f", "#f22", "#fa2",
    "#f2f", "#666", "#2f2", "#2af",
    "#22f", "#a00", "#060", "#006",
    "#000", "#000", "#000", "#000",
    "#000", "#000", "#000", "#000"
  ]


  const picoAudio = new PicoAudio()
  const pianoroll = ref()

  const pianorollSize = {
    width: 700,
    height: 500
  }
  const noteResolution = 20
  const noteHeight = 5
  const pianorollX = ref(-(pianorollSize.width / 2))
  const pianorollY = -pianorollSize.height - (noteHeight * 128 - pianorollSize.height) / 2

  const notes = {}
  const noteWidthRatio = ref(0)


  function play () {
    picoAudio.init()
    picoAudio.play()
  }

  function onMidiFileChanged (e) {
    const file = e.target.files[0]
    const reader = new FileReader()
    reader.readAsArrayBuffer(file)
    reader.onload = () => {
      const smf = new Uint8Array(reader.result as ArrayBuffer)
      const song = picoAudio.parseSMF(smf)
      picoAudio.setData(song)
      noteWidthRatio.value = (840 / song.header.resolution)
      renderPianoroll(song)
    };
  }

  function renderPianoroll(song) {
    song.channels.forEach((channel, channelIdx) => {
      channel.notes.forEach((note, idx)=> {
        // noteオブジェクトに固有のIDを追加する(イベント発火時にID入りの同じオブジェクトが渡される)
        note.id = `${channelIdx}-${idx}`

        let obj
        if (channelIdx === 9) {
          obj = document.createElementNS('http://www.w3.org/2000/svg', 'circle')

          obj.setAttribute('cx', note.start / noteResolution * noteWidthRatio.value)
          obj.setAttribute('cy', -(note.pitch - 0.5) * noteHeight);
          obj.setAttribute('r', noteHeight / 2);
          obj.setAttribute('stroke-width', 5)
        } else {
          obj = document.createElementNS('http://www.w3.org/2000/svg', 'polygon')

          const pitchBends = [
            ...note.pitchBend,
            {
              timing: note.stop,
              value: note.pitchBend.slice(-1)[0].value
            }
          ]
          const line = pitchBends.map((pitchBend) => {
            return {
              x: pitchBend.timing / noteResolution * noteWidthRatio.value,
              y: -(note.pitch + pitchBend.value) * noteHeight
            }
          })
          const points = [
            ...line.map((note) => `${note.x},${note.y}`),
            ...line.reverse().map((note) => `${note.x},${note.y + noteHeight}`)
          ]
          obj.setAttribute('points', points.join(' '))
          obj.setAttribute('stroke-width', 3)
        }

        obj.setAttribute('fill', ChannelColor[channelIdx])
        obj.setAttribute('opacity', note.velocity * 0.75)

        pianoroll.value.appendChild(obj)
        notes[note.id] = obj
      })
    })
  }


  onMounted(() => {
    const render = () => {
      const baseTime = picoAudio.states.isPlaying ? picoAudio.context.currentTime : picoAudio.states.stopTime
      const timing = picoAudio.getTiming(baseTime - picoAudio.states.startTime)

      if (timing) {
        pianorollX.value = (timing / noteResolution * noteWidthRatio.value) - (pianorollSize.width / 2)
      }

      window.requestAnimationFrame(render)
    }
    render()

    picoAudio.addEventListener('noteOn', (note) => {
      console.log(note)
      notes[note.id].setAttribute('opacity', note.velocity * 1.3)
      notes[note.id].setAttribute('stroke', ChannelColor[note.channel])
    })

    picoAudio.addEventListener('noteOff', (note) => {
      notes[note.id].setAttribute('opacity', note.velocity * 0.75)
      notes[note.id].setAttribute('stroke', 'none')
    })
  })
</script>
