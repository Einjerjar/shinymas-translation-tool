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
                <div class="d-block flex-grow-0 me-2 fw-bold">{{ i + 1 }}</div>
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
              #{{ currentLine + 1 }}
            </div>
            <div class="d-flex mb-2 align-items-center">
              <input v-model="currentItem.name" type="text" class="form-control">
              <input v-model="currentItem.id" type="text" class="form-control ms-2" disabled>
            </div>
            <textarea v-model="currentItem.text" rows="3" style="resize: none" class="form-control mb-2"
                      disabled></textarea>
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
import {onMounted, reactive, ref, watchEffect} from 'vue'
import Papa from 'papaparse'
import name_data from "../name_data";

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

    console.info('[Parsing name translation reference]')
    const parsedNameData = Papa.parse(name_data, {
      header: true,
      skipEmptyLines: true
    })

    const parsedNameDict = {}
    const nameOverrides = {}

    for (const i in parsedNameData.data) {
      const data = parsedNameData.data[i]
      parsedNameDict[data['name']] = data['trans']
    }
    console.info('[Done parsing name translation reference]')

    // console.log('pn', parsedNameData)
    // console.log('pd', parsedNameDict)

    // Not beautiful but works and much faster than adding it to textList and having to filter everytime the user saves
    const localNameRef = ref(['name'])

    // Default content, helps avoid errors, rather than having to add error handlers
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

    // Reusable stuff
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

    // Invisible file input for loading stuff
    const loadFileInput = document.createElement('input')
    loadFileInput.type = 'file'
    loadFileInput.addEventListener('change', () => {
      currentLine.value = 0
      inputFileName.value = loadFileInput.files[0].name

      console.info(`[Parsing ${loadFileInput.files[0].name}]`)
      Papa.parse(loadFileInput.files[0], {
        header: true,
        skipEmptyLines: true,
        complete: res => {
          textList.value = []
          localNameRef.value = []

          for (const i in res.data) {
            const data = res.data[i]
            textList.value.push(data)
            localNameRef.value.push(data.name)
          }

          // Automatically update the selection
          if (textList.value.length > 0) {
            editLine(0)
          }

          console.info(`[Done Parsing ${loadFileInput.files[0].name}]`)
        }
      })
    })

    const loadFile = () => {
      loadFileInput.click()
    }
    const downloadFile = () => {
      // Encodes the csv, then save via an invisible download link,
      //   might be a good idea to make a reusable one instead of creating a new one everytime
      const csv_data = encodeURIComponent(Papa.unparse(textList.value))
      const temp_link = document.createElement('a')
      temp_link.href = 'data:text/plain;charset=utf-8,' + csv_data
      temp_link.download = inputFileName.value
      temp_link.click()
    }

    const editLine = (line) => {
      if (!textList.value.hasOwnProperty(line)) return

      const currentName = localNameRef.value[line]

      const lineVal = textList.value[line]
      currentItem.id = lineVal.id

      // TODO: optimize
      // Got lazy on this one, can be optimized/shortened a lot
      // Checks if we have a local override for the name, then if we have a known translation,
      //   if none exists, then just use it as is
      currentItem.name = currentName === lineVal.name
          ? currentName in nameOverrides
              ? nameOverrides[currentName]
              : currentName in parsedNameDict
                  ? parsedNameDict[currentName]
                  : currentName
          : currentName
      currentItem.text = lineVal.text
      currentItem.trans = lineVal.trans

      currentLine.value = line
    }

    const revertLine = () => {
      if (!textList.value.hasOwnProperty(currentLine.value)) return

      const lineVal = textList.value[currentLine.value]
      console.log(lineVal)

      // TODO: optimize
      // Got lazy on this one, can be optimized/shortened a lot
      // Checks if we have a local override for the name, then if we have a known translation,
      //   if none exists, then just use it as is
      localNameRef.value[currentLine.value] = lineVal.name
      const currentName = localNameRef.value[currentLine.value]
      currentItem.name = currentName === lineVal.name
          ? currentName in nameOverrides
              ? nameOverrides[currentName]
              : currentName in parsedNameDict
                  ? parsedNameDict[currentName]
                  : currentName
          : currentName
      currentItem.trans = lineVal.trans
    }
    const saveLine = () => {
      if (!textList.value.hasOwnProperty(currentLine.value)) return

      localNameRef.value[[currentLine.value]] = currentItem.name
      textList.value[currentLine.value].trans = currentItem.trans
    }

    onMounted(() => {
      // Get the container, create a canvas for drawing, get it's context,
      //   create a new image to paste the canvas over to, image is used so we can
      //   apply css properties like img-fluid with ease
      const canvasContainer = document.getElementById('canvasContainer');
      const canvasRefImage = document.createElement('img')
      const canvasRef = document.createElement('canvas')
      canvasRefImage.classList.add('img-fluid')

      // Allow simple interactivity
      canvasRefImage.addEventListener('click', () => {
        if (currentLine.value < textList.value.length - 1) {
          currentLine.value += 1
          editLine(currentLine.value)
        }
      })

      // Just some canvas stuff
      canvasRef.width = canvasOptions.width
      canvasRef.height = canvasOptions.height
      canvasContainer.appendChild(canvasRefImage)

      const ctx = canvasRef.getContext('2d');

      ctx.textBaseline = 'top'
      ctx.fillStyle = '#585858'

      // Simple asset loader
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

      // Draws the canvas, sufficient for the time being as the canvas isn't animated
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

          // Might be costy, but as said before, sufficient for now
          canvasRefImage.src = canvasRef.toDataURL('image/png')
        }
      }

      // Asset loading routine
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

      // Update canvas for every changes on the ref vars, truly, vue3 is just <3 <3 (lol)
      watchEffect(() => {
        updateCanvas()
        if (autosave) {
          saveLine()
        }
      })

      // Update routine for the custom name overrides
      watchEffect(() => {
        const line = currentLine.value

        const orig = textList.value[line].name
        const local = localNameRef.value[line]
        const nameDict = orig in parsedNameDict ? parsedNameDict[orig] : orig

        if (local !== nameDict && local !== orig) {
          nameOverrides[orig] = local
        }
      })

      // Initial canvas draw
      updateCanvas()
    })

    return {
      inputFileName,
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
