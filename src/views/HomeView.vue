<template>
  <div
    v-for="(video, index) in videos"
    :key="index"
    style="margin-bottom:20px;"
  >
    <input
      type="file"
      @change="selectVideo($event, index)"
      accept=".mp4,.mov,.mkv"
    />

    <select v-model="video.videoResolution">
      <option disabled value="">
        Select Resolution
      </option>

      <option value="1920">
        1920 x 1080
      </option>

      <option value="1280">
        1280 x 720
      </option>
    </select>
  </div>

  <button @click="addMore">
    Add More
  </button>

  <button
    @click="compressVideo"
    :disabled="!isLoaded"
  >
    {{ isLoaded ? 'Compress Videos' : 'Loading FFmpeg...' }}
  </button>

  <p>{{ statusMessage }}</p>

<div
  v-for="(video, index) in compressedVideos"
  :key="index"
  style="margin-bottom:20px;border:1px solid #ccc;padding:10px;"
>
  <h3>{{ video.fileName }}</h3>

  <h4>MP4</h4>

  <video
    :src="video.mp4"
    controls
    width="400"
  ></video>

  <br>

  <a
    :href="video.mp4"
    :download="`${video.fileName}.mp4`"
  >
    Download MP4
  </a>

  <hr>

  <h4>WEBM</h4>

  <video
    :src="video.webm"
    controls
    width="400"
  ></video>

  <br>

  <a
    :href="video.webm"
    :download="`${video.fileName}.webm`"
  >
    Download WEBM
  </a>

  <hr>

  <h4>OGG AUDIO</h4>

  <audio
    :src="video.ogg"
    controls
  ></audio>

  <br>

  <a
    :href="video.ogg"
    :download="`${video.fileName}.ogg`"
  >
    Download OGG
  </a>
</div>
</template>

<script>
import { FFmpeg } from '@ffmpeg/ffmpeg'
import { fetchFile, toBlobURL } from '@ffmpeg/util'
import { markRaw } from 'vue'

export default {
  data() {
    return {
      ffmpeg: null,

      isLoaded: false,

      statusMessage: 'Loading FFmpeg...',

      videos: [
        {
          file: null,
          fileName: '',
          videoResolution: ''
        }
      ],

      compressedVideos: []
    }
  },

  async mounted() {
    this.ffmpeg = markRaw(new FFmpeg())

    await this.loadFFmpeg()
  },

  methods: {
    addMore() {
      this.videos.push({
        file: null,
        fileName: '',
        videoResolution: ''
      })
    },

    async loadFFmpeg() {
      const baseURL =
        'https://cdn.jsdelivr.net/npm/@ffmpeg/core@0.12.6/dist/esm'

      this.ffmpeg.on('log', ({ message }) => {
        console.log(message)
      })

      await this.ffmpeg.load({
        coreURL: await toBlobURL(
          `${baseURL}/ffmpeg-core.js`,
          'text/javascript'
        ),

        wasmURL: await toBlobURL(
          `${baseURL}/ffmpeg-core.wasm`,
          'application/wasm'
        )
      })

      this.isLoaded = true
      this.statusMessage = 'FFmpeg Loaded'
    },

    selectVideo(e, index) {
      const file = e.target.files[0]

      if (!file) return

      const extension =
        file.name.split('.').pop().toLowerCase()

      if (!['mp4', 'mov', 'mkv'].includes(extension)) {
        alert('Invalid file type')
        return
      }

      const fileName = file.name.substring(0, file.name.lastIndexOf('.'))

      this.videos[index].file = file
      this.videos[index].fileName = fileName
    },

    async compressVideo() {
      try {
        this.compressedVideos = []

        for (let i = 0; i < this.videos.length; i++) {
          const video = this.videos[i]

          if (!video.file) continue

          if (!video.videoResolution) {
            alert(`Select resolution for video ${i + 1}`)
            return
          }

          this.statusMessage =
            `Compressing ${i + 1} / ${this.videos.length}`

          const extension =
            video.file.name.split('.').pop()

          const inputFile =
            `input_${i}.${extension}`

          const mp4Output =
            `output_${i}.mp4`

          const oggOutput =
            `output_${i}.ogg`

          const webmOutput =
            `output_${i}.webm`

          // write source file
          await this.ffmpeg.writeFile(
            inputFile,
            await fetchFile(video.file)
          )

          /* MP4 */
          await this.ffmpeg.exec([
            '-i',
            inputFile,
            '-vf',
            `scale=${video.videoResolution}:-2`,
            '-c:v',
            'libx264',
            '-crf',
            '35',
            '-preset',
            'ultrafast',
            '-c:a',
            'copy',
            '-movflags',
            '+faststart',
            mp4Output
          ])

          /* OGG */
          await this.ffmpeg.exec([
            '-i',
            inputFile,
            '-vn',
            '-acodec',
            'libvorbis',
            oggOutput
          ])

          /* WEBM */
          await this.ffmpeg.exec([
            '-i',
            inputFile,

            '-vf',
            `scale=${video.videoResolution}:-2`,

            '-c:v',
            'libvpx',

            '-crf',
            '35',

            '-b:v',
            '0',

            '-deadline',
            'realtime',

            '-cpu-used',
            '8',

            '-c:a',
            'libvorbis',

            webmOutput
          ])

          /* READ MP4 */
          const mp4Data =
            await this.ffmpeg.readFile(mp4Output)

          const mp4Blob = new Blob(
            [mp4Data.buffer],
            { type: 'video/mp4' }
          )

          const mp4Url =
            URL.createObjectURL(mp4Blob)

          /* READ OGG */
          const oggData =
            await this.ffmpeg.readFile(oggOutput)

          const oggBlob = new Blob(
            [oggData.buffer],
            { type: 'audio/ogg' }
          )

          const oggUrl =
            URL.createObjectURL(oggBlob)

          /* READ WEBM */
          const webmData =
            await this.ffmpeg.readFile(webmOutput)

          const webmBlob = new Blob(
            [webmData.buffer],
            { type: 'video/webm' }
          )

          const webmUrl =
            URL.createObjectURL(webmBlob)

          this.compressedVideos.push({
            fileName: video.fileName,

            mp4: mp4Url,
            ogg: oggUrl,
            webm: webmUrl
          })

          // cleanup
          await this.ffmpeg.deleteFile(inputFile)

          await this.ffmpeg.deleteFile(mp4Output)
          await this.ffmpeg.deleteFile(oggOutput)
          await this.ffmpeg.deleteFile(webmOutput)
        }

        this.statusMessage =
          'Compression complete!'
      } catch (err) {
        console.error(err)
        this.statusMessage =
          'Compression failed'
      }
    }
  }
}
</script>