<template>
  <div class="convo-edit container-fluid p-4 h-100">
    <div class="row h-100">
      <div class="col-md-3 order-last order-md-first h-100 mb-2">
        <div class="card h-100">
          <div class="card-header d-flex">
            <input v-model="inputFileName" type="text" class="form-control me-auto">
            <button class="btn" @click="loadFile">
              <i class="bi-folder2-open"></i>
            </button>
            <button class="btn" @click="downloadFile">
              <i class="bi-download"></i>
            </button>
          </div>
          <ul class="list-group list-group-flush overflow-auto">
            <li class="list-group-item" v-for="(v, i) in textList" :key="i">
              <div class="d-flex">
                <div class="d-block flex-grow-0 me-2 fw-bold">{{i+1}}</div>
                <div>{{ v.trans }}</div>
                <div class="ms-auto flex-grow-0">
                  <button class="btn d-block" @click="editLine(i)">
                    <i class="bi-pencil"></i>
                  </button>
                </div>
              </div>
            </li>
          </ul>
        </div>
      </div>
      <div class="col-md-9">
        <div class="card bg-light mb-2">
          <div class="card-header d-flex">
            <div class="h5 me-auto">SHINYMAS Translation Helper</div>
            <a href="https://github.com/Einjerjar/shinymas-translation-tool" class="btn" title="Code on github">
              <i class="bi-github"></i>
            </a>
          </div>
          <div class="card-body">
            <div id="canvasContainer" class="d-flex justify-content-center">
            </div>
            <hr>
            <div class="mb-2 fw-bold">
              #{{currentLine + 1}}
            </div>
            <div class="d-flex mb-2 align-items-center">
              <input v-model="currentItem.name" type="text" class="form-control" disabled>
              <input v-model="currentItem.id" type="text" class="form-control ms-2" disabled>
            </div>
            <textarea v-model="currentItem.text" rows="3" style="resize: none" class="form-control mb-2" disabled></textarea>
            <textarea v-model="currentItem.trans" rows="3" style="resize: none" class="form-control"></textarea>
            <div class="d-flex pt-2 align-items-center">
              <div class="form-check">
                <input id="autosaveCheckbox" type="checkbox" class="form-check-input" v-model="autosave">
                <label for="autosaveCheckbox" class="form-check-label">Autosave</label>
              </div>
              <button class="btn btn-warning ms-auto me-2" @click="revertLine" :disabled="autosave">
                Revert
              </button>
              <button class="btn btn-primary" @click="saveLine" :disabled="autosave">
                Save
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import {ref, watchEffect, onMounted, reactive} from 'vue'
import Papa from 'papaparse'

export default {
  name: "ConvoEdit",
  setup() {
    const inputFileName = ref('Filename.csv')
    const currentItem = reactive({
      id: '0',
      name: 'Name',
      text: '翻訳されていないテキスト', // Courtesy of google trans, don't blame me lol
      trans: 'Translated text',
    })

    const textList = ref([
      {
        id: '0',
        name: 'Name',
        text: '翻訳されていないテキスト', // Courtesy of google trans, don't blame me lol
        trans: 'Translated text'
      }
    ])

    // Make a normal var to avoid the .value extensions, and just add a computed value for rendering?
    const currentLine = ref(0)
    const autosave = ref(false)

    const canvasOptions = {
      // width: 840,
      // height: 162,
      width: 722,
      height: 146,
      // font: 'FOT-Humming Std',
      font: 'Humming-C',
      nameText: {
        x: 43,
        y: 9,
        size: 26,
        font: ''
      },
      contentText: {
        x: 31,
        y: 56,
        size: 22,
        gap: 8,
        font: '',
        letterOffset: 0.42,
      },
      detailA: {
        x: 680,
        y: 120
      }
    }
    canvasOptions.nameText.font = `${canvasOptions.nameText.size}px ${canvasOptions.font}`
    canvasOptions.contentText.font = `${canvasOptions.contentText.size}px ${canvasOptions.font}`

    const loadFileInput = document.createElement('input')
    loadFileInput.type = 'file'
    loadFileInput.addEventListener('change', () => {
      currentLine.value = 0
      inputFileName.value = loadFileInput.files[0].name
      Papa.parse(loadFileInput.files[0], {
        header: true,
        skipEmptyLines: true,
        complete: res => {
          textList.value = []
          for (const i in res.data) {
            const data = res.data[i]
            textList.value.push(data)
          }
          if (textList.value.length > 0) {
            editLine(0)
          }
        }
      })
    })

    const loadFile = () => {
      loadFileInput.click()
    }
    const downloadFile = () => {
      const csv_data = encodeURIComponent(Papa.unparse(textList.value))
      const temp_link = document.createElement('a')
      temp_link.href = 'data:text/plain;charset=utf-8,' + csv_data
      temp_link.download = inputFileName.value
      temp_link.click()
    }

    const editLine = (line) => {
      if (!textList.value.hasOwnProperty(line)) return

      const lineVal = textList.value[line]
      currentItem.id = lineVal.id
      currentItem.name = lineVal.name
      currentItem.text = lineVal.text
      currentItem.trans = lineVal.trans

      currentLine.value = line
    }

    const revertLine = () => {
      if (!textList.value.hasOwnProperty(currentLine.value)) return

      const lineVal = textList.value[currentLine.value]
      console.log(lineVal)
      currentItem.trans = lineVal.trans
    }
    const saveLine = () => {
      if (!textList.value.hasOwnProperty(currentLine.value)) return

      textList.value[currentLine.value].trans = currentItem.trans
    }

    onMounted(() => {
      const canvasContainer = document.getElementById('canvasContainer');
      const canvasRefImage = document.createElement('img')
      const canvasRef = document.createElement('canvas')
      canvasRefImage.classList.add('img-fluid')

      canvasRef.addEventListener('click', () => {
        if (currentLine.value < textList.value.length-1) {
          currentLine.value += 1
          editLine(currentLine.value)
        }
      })

      canvasRef.width = canvasOptions.width
      canvasRef.height = canvasOptions.height
      canvasContainer.appendChild(canvasRefImage)

      const ctx = canvasRef.getContext('2d');

      ctx.textBaseline = 'top'
      ctx.fillStyle = '#585858'

      let allAssetsReady = false

      let loadedAssets = 0
      const imageAssetFiles = {
        bgImageR: 'img/text-bg-r.png',
        // bgImageG: 'img/text-bg-g.png',
        // bgImageB: 'img/text-bg-b.png',
        detailA: 'img/detail-a.png'
      }
      const imageAssets = {}
      const requiredAssets = Object.keys(imageAssetFiles).length

      const updateCanvas = () => {
        ctx.clearRect(0, 0, canvasOptions.width, canvasOptions.height)
        if (allAssetsReady) {
          ctx.drawImage(imageAssets['bgImageR'], 0, 0)
        }

        ctx.font = canvasOptions.nameText.font;
        ctx.fillText(currentItem.name, canvasOptions.nameText.x, canvasOptions.nameText.y)
        ctx.font = canvasOptions.contentText.font
        const texts = (currentItem.trans + '').split('\n')
        for (const i in texts) {
          const ix = parseInt(i)
          const text = texts[i]
          const baseX = canvasOptions.contentText.x
          const baseY = canvasOptions.contentText.y + (ix * (canvasOptions.contentText.size + canvasOptions.contentText.gap))
          let deltaX = 0
          for (const j in text) {
            const letter = text[j]
            ctx.fillText(letter, baseX + deltaX, baseY)
            deltaX += ctx.measureText(letter).width + canvasOptions.contentText.letterOffset
          }

          if (allAssetsReady) {
            ctx.drawImage(imageAssets['detailA'], canvasOptions.detailA.x, canvasOptions.detailA.y)
          }

          canvasRefImage.src = canvasRef.toDataURL('image/png')
        }
      }

      for (const assetName in imageAssetFiles) {
        const assetUrl = imageAssetFiles[assetName]
        const assetImage = document.createElement('img');
        assetImage.addEventListener('load', () => {
          loadedAssets += 1
          if (loadedAssets === requiredAssets) {
            allAssetsReady = true
            updateCanvas()
          }
        })
        assetImage.src = assetUrl
        imageAssets[assetName] = assetImage
      }

      watchEffect(() => {
        updateCanvas()
        if (autosave) {
          saveLine()
        }
      })

      updateCanvas()
    })

    return {inputFileName,
      currentItem,
      textList,
      currentLine,
      autosave,
      loadFile,
      downloadFile,
      editLine,
      revertLine,
      saveLine
    };
  }
}
</script>

<style scoped>

</style>
