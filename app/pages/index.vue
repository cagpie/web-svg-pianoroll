<template>
  <div>
    <div
      :style="{position: 'relative', width: `${pianorollSize.width}px`, height: `${pianorollSize.height}px`}"
    >
      <svg
        ref="pianoroll"
        style="position: absolute, top: 0"
        :width="pianorollSize.width"
        :height="pianorollSize.height"
        :viewBox="`${pianorollX} ${pianorollY} ${pianorollSize.width} ${pianorollSize.height}`"
        xmlns="http://www.w3.org/2000/svg"
      ></svg>
      <div :style="{position: 'absolute', width: '1px', height: `${pianorollSize.height}px`, top: '0', left: `${pianorollSize.width / 2}px`, backgroundColor: '#888', opacity: '0.2', display: isDisplayCenterLine ? 'block' : 'none'}"></div>
      <div :style="{position: 'absolute', background: pianorollBackground, top: '0', left: '0', right: '0', bottom: '0', zIndex: -1}"></div>
      <svg
        ref="circleOfFifth"
        :style="{position: 'absolute', left: '70px', bottom: '70px', zIndex: 5, display: isDisplayCircleOfFifth ? 'block' : 'none'}"
        :width="200"
        :height="200"
        :viewBox="`-100 -100 200 200`"
        xmlns="http://www.w3.org/2000/svg"
      >
        <polygon ref="cofPolygon" fill="rgba(200, 255, 155, 0.3)"></polygon>
        <text ref="cofRootName" x="-15" y="0" font-size="18" fill="#9a4" stroke="#9a4"></text>
      </svg>
    </div>
    <div style="margin-top: 20px">
      <div>
        <input type="file" accept="audio/midi" @change="onMidiFileChanged" />
        <button @click="play">play</button>
        <button @click="picoAudio.pause">pause</button>
        <button @click="picoAudio.initStatus">init</button>
      </div>
      <div>
        <div>
          width <input type="number" v-model="pianorollSize.width" />
          height <input type="number" v-model="pianorollSize.height" />
        </div>
        <div>
          background <input type="text" v-model="pianorollBackground" />
          <div>
            e.g.
            <code style="display: block">#f2f2f2</code>
            <code style="display: block">radial-gradient(#ffffff, #aaaaaa)</code>
            <code style="display: block">url('https://cagpie.net/data/img/kawaii.png')</code>
          </div>
        </div>
        <div>
          center line
          <input type="checkbox" v-model="isDisplayCenterLine">
        </div>
        <div>
          circle of fifth
          <input type="checkbox" v-model="isDisplayCircleOfFifth">
        </div>
      </div>
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
    "#22f", "#a00", "#060", "#006"
  ]

  const picoAudio = new PicoAudio()
  const pianoroll = ref()

  const pianorollSize = reactive({
    width: 1131,
    height: 636
  })
  const noteResolution = 20
  const noteHeight = 5
  const pianorollX = ref(-(pianorollSize.width / 2))
  const pianorollY = computed(() => -pianorollSize.height - (noteHeight * 128 - pianorollSize.height) / 2)
  const pianorollBackground = ref('#22113f')
  const isDisplayCenterLine = ref(true)
  const isDisplayCircleOfFifth = ref(true)


  const notes = {}
  const noteWidthRatio = ref(0)

  const playingNotes: Array<number> = []

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
      // 無理矢理曲長さを10秒伸ばす
      picoAudio.tempoTrack.slice(-1)[0].time += 10

      renderNotes(song)
    };
  }

  function renderNotes(song) {
    song.channels.forEach((channel, channelIdx) => {
      channel.notes.forEach((note, idx)=> {
        // noteオブジェクトに固有のIDを追加する(イベント発火時にID入りの同じオブジェクトが渡される)
        note.id = `${channelIdx}-${idx}`

        let obj
        if (channelIdx === 9) {
          obj = document.createElementNS('http://www.w3.org/2000/svg', 'circle')

          obj.setAttribute('cx', note.start / noteResolution * noteWidthRatio.value)
          obj.setAttribute('cy', -(note.pitch - 0.5) * noteHeight)
          obj.setAttribute('r', noteHeight / 2)
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


        obj.setAttribute('class', `ch-${channelIdx}`)
        // 重くなるので使用を避ける
        // obj.style.mixBlendMode = 'multiply'

        pianoroll.value.appendChild(obj)
        notes[note.id] = obj
      })
    })
  }

  // circleOfFifth = cof
  const circleOfFifth = ref()
  const cofRootName = ref()
  const cofPolygon = ref()

  const CHORDS = ["C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A", "A#", "B"];
  const C_CHORDS =	{
    "C":     [0,4,7],    "C7":     [0,4,7,10],
    "Cm":    [0,3,7],    "Cm7":    [0,3,7,10],
    "CM7":   [0,4,7,11], "CmM7":   [0,3,7,11],
    "Csus4": [0,5,7],    "C7sus4": [0,5,7,10],
    "Cdim":  [0,3,6,9],  "Cm7-5":  [0,3,6,10],
    "Caug":  [0,4,8],    "Cadd9":  [0,2,7],
    "C6":    [0,4,7,9],  "Cm6":    [0,3,7,9]
  }

  const cofWidth = 80
  const cof = [0, 7, 2, 9, 4, 11, 6, 1, 8, 3, 10, 5]
	const cofContents = ['C ', 'G ', 'D ', 'A ', 'E ', 'B ', 'F#', 'C#', 'G#', 'D#', 'A#', 'F ']
  const cofContentTexts = []

  function sort (a, b) {
    if (a < b) return -1
    if (a > b) return 1
    return 0
  }

  function getCofPosition(idx, width = cofWidth) {
    return {
      x: Math.cos(2 * Math.PI / 12 * idx - Math.PI / 2) * width,
      y: Math.sin(2 * Math.PI / 12 * idx - Math.PI / 2) * width
    }
  }

  function renderCircleOfFifthsBackground () {
    const fontSize = 15

    const background = document.createElementNS('http://www.w3.org/2000/svg', 'circle')
    background.setAttribute('cx', 0)
    background.setAttribute('cy', 0)
    background.setAttribute('r', cofWidth + (fontSize * 0.7))
    background.setAttribute('fill', 'rgba(255, 255, 255, 0.2)')
    circleOfFifth.value.appendChild(background)

    cofContents.forEach((content, idx) => {
      const position = getCofPosition(idx)
      const text = document.createElementNS('http://www.w3.org/2000/svg', 'text')
      text.setAttribute('x', position.x - fontSize / 2)
      text.setAttribute('y', position.y + fontSize / 2)
      text.setAttribute('font-size', fontSize)
      text.appendChild(document.createTextNode(content))
      circleOfFifth.value.appendChild(text)
      cofContentTexts.push(text)
    })
  }

  function renderCircleOfFifthsContent () {
    let rootNote = 128;
    const playingNotesMod = playingNotes.map((note)=> {
      // コードのルート音推定
      if (note < rootNote) {
        rootNote = note
      }
      // ド～シのみに絞る
      return (note % 12)
    }).filter((note, idx, arr)=> {
      // 重複をはじく
      return arr.indexOf(note) == idx
    }).sort(sort)

    const playingNotePositions = playingNotesMod.map((note)=> {
      return cof[note]
    }).sort(sort).map((idx) => {
      const position = getCofPosition(idx, cofWidth * 0.9)
      return `${position.x},${position.y}`
    })

    // ルート音を光らせる
    cofContentTexts.forEach((text, idx) => {
      if (rootNote !== 128 && idx === cof[rootNote % 12]) {
        return text.setAttribute('stroke', '#9a4')
      }

      text.setAttribute('stroke', '#222')
    })

    // 図形描画
    cofPolygon.value.setAttribute('points', playingNotePositions.join(' '))

    // コード描画
    let currentCodeName = '';
    for (let i = 0; i < 12; i++) {
      Object.keys(C_CHORDS).forEach((code)=> {
        // コードのリストと合致する音を取り出す
        const _list = playingNotesMod.filter((note)=> {
          // 一音ずつずらしたコードと判定
          return C_CHORDS[code].map(n => (n + i) % 12).indexOf(note) != -1
        });
        if ((_list.length === C_CHORDS[code].length) ||
          (_list.length >= 4)) {
            currentCodeName = CHORDS[i] + code.slice(1);
        }
      });

      if (i === rootNote % 12) {
        cofRootName.value.textContent = currentCodeName
      }
    }
  }

  onMounted(() => {
    const render = () => {
      const baseTime = picoAudio.states.isPlaying ? picoAudio.context.currentTime : picoAudio.states.stopTime
      const timing = picoAudio.getTiming(baseTime - picoAudio.states.startTime)

      if (timing) {
        pianorollX.value = (timing / noteResolution * noteWidthRatio.value) - (pianorollSize.width / 2)
      }

      renderCircleOfFifthsContent()

      window.requestAnimationFrame(render)
    }
    render()

    picoAudio.addEventListener('noteOn', (note) => {
      notes[note.id].setAttribute('opacity', note.velocity * 1.1)
      notes[note.id].setAttribute('stroke', ChannelColor[note.channel])

      if (note.channel !== 9) {
        playingNotes.push(note.pitch)
      }
    })

    picoAudio.addEventListener('noteOff', (note) => {
      notes[note.id].setAttribute('opacity', note.velocity * 0.75)
      notes[note.id].setAttribute('stroke', 'none')

      if (note.channel !== 9) {
        playingNotes.some((pitch, idx)=> {
          if (note.pitch === pitch) {
            playingNotes.splice(idx, 1)
            return true
          }
        })
      }
    })

    renderCircleOfFifthsBackground()
  })
</script>
