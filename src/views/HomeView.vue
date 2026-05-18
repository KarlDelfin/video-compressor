<template>
  <div class="form_con" v-for="(video, index) in videos" :key="index">
    <input
      class="file"
      type="file"
      @change="selectVideo($event, index)"
      accept=".mp4,.mov,.mkv"/>

    <select class="select" v-model="video.videoResolution">
      <option disabled value=""> Select Resolution </option>
      <option value="1920"> 1920 x 1080 </option>
      <option value="1280"> 1280 x 720 </option>
    </select>

    <button class="remove" @click="removeVideo(index)" v-if="videos.length > 1">&times;</button>
  </div>

  <button class="add_more" @click="addMore">Add More</button>
  <hr>
  <button
    class="compress"
    @click="compressVideo"
    :disabled="!isLoaded || videos.length === 0 || videos.some(video => !video.file || !video.videoResolution)">
    {{ isLoaded ? 'Compress Videos' : 'Loading FFmpeg...' }}
  </button>

  <p>{{ statusMessage }}</p>

  <!-- COMPRESSED VIDEOS -->
  <div v-for="(video, index) in compressedVideos" :key="index" >
    <div class="converted_video_info">
      <h3>{{ video.fileName }}</h3>
    </div>
    <div class="converted_video_con">
      <div>
        <h4>MP4</h4>
        <video :src="video.mp4" controls></video>
        <a :href="video.mp4" :download="`${video.fileName}.mp4`">Download MP4</a>
      </div>

      <div>
        <h4>WEBM</h4>
        <video :src="video.webm" controls></video>
        <a :href="video.webm" :download="`${video.fileName}.webm`">Download WEBM</a>
      </div>

      <div>
        <h4>OGG AUDIO</h4>
        <audio :src="video.ogg" controls ></audio>
        <a :href="video.ogg" :download="`${video.fileName}.ogg`">Download OGG</a>
      </div>
    </div>
  </div>
</template>

<script>
import { FFmpeg } from '@ffmpeg/ffmpeg'
import { fetchFile, toBlobURL } from '@ffmpeg/util'
import { markRaw } from 'vue'
import ffmpegCoreJs from '@/assets/js/ffmpeg-core.js?url'
import ffmpegCoreWasm from '@/assets/js/ffmpeg-core.wasm?url'

export default {
  data() {
    return {
      ffmpeg: null,

      isLoaded: false,
      progress: 0,
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

  methods: {
    removeVideo(index) {
      this.videos.splice(index, 1)
    },
    addMore() {
      this.videos.push({
        file: null,
        fileName: '',
        videoResolution: ''
      })
    },

    async loadFFmpeg() {

      this.ffmpeg.on('log', ({ message }) => {
        console.log(message)
      })

      this.ffmpeg.on('progress', ({ progress, time }) => {
        this.progress = Math.round(progress * 100)

        this.statusMessage = `Compressing... ${this.progress}%`
      })

      await this.ffmpeg.load({
        coreURL: ffmpegCoreJs,
        wasmURL: ffmpegCoreWasm
      })

      this.isLoaded = true
      this.statusMessage = 'FFmpeg Loaded'
    },

    selectVideo(e, index) {
      const file = e.target.files[0]

      if (!file) return

      const extension = file.name.split('.').pop().toLowerCase()

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

          this.statusMessage = `Compressing ${i + 1} / ${this.videos.length}`

          const extension = video.file.name.split('.').pop()

          const inputFile = `input_${i}.${extension}`

          const mp4Output = `output_${i}.mp4`

          const oggOutput = `output_${i}.ogg`

          const webmOutput = `output_${i}.webm`

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
          const mp4Data = await this.ffmpeg.readFile(mp4Output)

          const mp4Blob = new Blob( [mp4Data.buffer], { type: 'video/mp4' } )

          const mp4Url = URL.createObjectURL(mp4Blob)

          /* READ OGG */
          const oggData = await this.ffmpeg.readFile(oggOutput)

          const oggBlob = new Blob( [oggData.buffer], { type: 'audio/ogg' } )

          const oggUrl = URL.createObjectURL(oggBlob)

          /* READ WEBM */
          const webmData = await this.ffmpeg.readFile(webmOutput)

          const webmBlob = new Blob( [webmData.buffer], { type: 'video/webm' } )

          const webmUrl = URL.createObjectURL(webmBlob)

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

        this.statusMessage = 'Compression complete!'
      } catch (err) {
        console.error(err)
        this.statusMessage = 'Compression failed'
      }
    }
  },

  async mounted() {
    this.ffmpeg = markRaw(new FFmpeg())
    await this.loadFFmpeg()
    console.log(this.videos.length)
  },
}
</script>

<style scoped>
  hr {color: #eee; }
  .form_con { display: flex; align-items: center; gap: 15px; margin-bottom: 20px; }
  
  .file {border: 1px solid #d6d6d6; border-radius: 7px; background-color: #fff; color: #444; box-shadow: none; padding: 5px 10px;}
  
  .select {border: 1px solid #d6d6d6; border-radius: 7px; background-color: #fff; color: #444; box-shadow: none; padding: 9px 10px;}
  
  .remove {background: #ef476f; font-size: 16px; color: #fff; text-align: center; position: relative; border-radius: 100px; font-weight: 500; border: none; cursor: pointer; transition: .3s ease; height: 35px; width: 35px; display: flex; justify-content: center; align-items: center;}
  .remove:hover {background: #fff; color:  	#ef476f; border: 1px solid  	#ef476f}
  
  .add_more {background: #118ab2; font-size: 14px; color: #fff; text-align: center; position: relative; font-weight: 500; border: none; cursor: pointer; padding: 10px; border-radius: 5px; transition: .3s ease;}
  .add_more:hover {background: #fff; color:  	#118ab2; border: 1px solid  	#118ab2}

  .compress {display: block; background: #073b4c; font-size: 14px; color: #fff; text-align: center; position: relative; font-weight: 500; border: none; cursor: pointer; padding: 10px; border-radius: 5px; transition: .3s ease;}
  .compress:hover {background: #fff; color:  	#073b4c; border: 1px solid  	#073b4c}
  .converted_video_info { text-align: center; }
  button:disabled { background-color: #cccccc; color: #666666; border: 1px solid #999999; cursor: not-allowed; opacity: 1; }
  
  .converted_video_con { display: flex; width: 100%; gap: 25px; }
  .converted_video_con div { width: 100%; border: 1px solid #e1e1e1; padding: 20px; display: flex; justify-content: space-between; align-items: center; flex-direction: column; border-radius: 10px; gap: 20px; }
  .converted_video_con div video {width: 100%;}
</style>