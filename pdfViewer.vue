<template>
  <div>
    <div class="tool-wrapper">
      <div class="tool-box">
        <span v-for="(item, i) in toolItems" :key="i">
          <!-- scrawl popover -->
          <a-popover
            v-if="item.type === 'Scrawl' && !readonly"
            placement="bottomLeft"
            trigger="click"
            :visible="item.popOpen"
            overlayClassName="font-size-color-wrapper"
          >
            <div slot="content" style="width: 200px">
              <div class="font-size-color-box">
                <span class="label-box">颜色</span>
                <colorPicker v-model="annotation.color" />
              </div>
              <div v-if="scrawlType === 'Text'" class="font-size-color-box">
                <span class="label-box">字号</span>
                <a-slider v-model="annotation.fontSize" :max="30" style="width: 100px" />
              </div>
              <div v-else class="font-size-color-box">
                <span class="label-box">粗细</span>
                <a-slider v-model="annotation.lineWidth" :max="10" style="width: 100px" />
              </div>
              <a-divider style="margin: 10px 0" />
              <div
                v-for="(o, k) in item.sub"
                :key="k"
                direction="vertical"
                class="sub-box font-size-color-box"
                @click="() => scrawlClick(i, k)"
                :class="o.checked && 'checked'"
              >
                <span class="label-box">{{ o.text }}</span>
                <a-icon :type="o.icon"></a-icon>
              </div>
            </div>
            <template slot="title">
              <div style="text-align: end">
                <a-icon type="close" @click="closePop(i)" />
              </div>
            </template>
            <a-icon :type="item.icon" :class="item.checked && 'checked'" @click="changeAnnotationType(item)"></a-icon>
          </a-popover>

          <a-icon
            v-if="item.type === 'Empty' || item.type === 'Download'"
            :type="item.icon"
            @click="changeAnnotationType(item)"
            :class="item.checked && 'checked'"
          ></a-icon>
        </span>
      </div>
      <!-- pagination box -->
      <div class="pagination-box">
        <a-pagination
          simple
          v-model="pagination.current"
          :total="pagination.total"
          :pageSize="1"
          @change="pageLocated"
        />
      </div>
      <!-- zoomin and zoomout box -->
      <div class="zoom-box">
        <a @click="zoomOut"><a-icon type="zoom-out" /></a>
        <a-select v-model="zoomNumber" @change="zoomChange">
          <a-select-option :value="50"> 50% </a-select-option>
          <a-select-option :value="100"> 100% </a-select-option>
          <a-select-option :value="150"> 150% </a-select-option>
          <a-select-option :value="200"> 200% </a-select-option>
        </a-select>
        <a @click="zoomIn"><a-icon type="zoom-in" /></a>
      </div>
    </div>

    <a-row class="pdf-wrapper">
      <a-col :span="19" class="pdf-box" style="position: relative">
        <div id="pdfContainer"></div>

        <!-- text create -->
        <a-input
          class="editor-input"
          v-if="inputTextValueVisible && annotationMode === 'Scrawl' && scrawlType === 'Text'"
          v-model="annotation.textValue"
          placeholder="请输入"
          :style="{
            left: annotation.x + 'px',
            top: annotation.yEditor + 'px',
            fontSize: annotation.fontSize + 'px',
            color: annotation.color,
          }"
          @keydown="enterKey"
        />

        <!-- rectangle virtual box -->
        <section
          v-if="scrawlType === 'Rectangle' && isDrawing"
          :style="{
            position: 'absolute',
            left: Math.min(annotation.x, annotation.endX) + 'px',
            top: Math.min(annotation.absoluteY, annotation.absoluteEndY) + 'px',
            width: Math.abs(annotation.endX - annotation.x) + 'px',
            height: Math.abs(annotation.endY - annotation.y) + 'px',
            border: '2px dashed black',
            pointerEvents: 'none',
          }"
        />

        <!-- elliptic virtual box -->
        <section
          v-if="scrawlType === 'Elliptic' && isDrawing"
          :style="{
            position: 'absolute',
            left: Math.min(annotation.x, annotation.endX) + 'px',
            top: Math.min(annotation.absoluteY, annotation.absoluteEndY) + 'px',
            width: Math.abs(annotation.endX - annotation.x) + 'px',
            height: Math.abs(annotation.endY - annotation.y) + 'px',
            border: '2px dashed black',
            pointerEvents: 'none',
            borderRadius: '50%',
          }"
        />

        <!-- straight line virtual line -->
        <section
          v-if="scrawlType === 'StraightLine' && isDrawing"
          :style="{
            position: 'absolute',
            left: Math.min(annotation.x, annotation.endX) + 'px',
            top: Math.min(annotation.absoluteY, annotation.absoluteEndY) + 'px',
            width: Math.abs(annotation.endX - annotation.x) + 'px',
            border: '2px solid black',
            pointerEvents: 'none',
          }"
        />

        <!-- Wave line virtual line -->
        <section
          v-if="scrawlType === 'WaveLine' && isDrawing"
          :style="{
            position: 'absolute',
            left: Math.min(annotation.x, annotation.endX) + 'px',
            top: Math.min(annotation.absoluteY, annotation.absoluteEndY) + 'px',
            width: Math.abs(annotation.endX - annotation.x) + 'px',
            border: '2px dashed black',
            pointerEvents: 'none',
          }"
        />

        <section
          v-if="focusAnnotation.x"
          class="focus-section"
          :style="{
            left: focusAnnotation.x - 5 + 'px',
            top: focusAnnotation.absoluteY - 5 + 'px',
            width: Math.abs(focusAnnotation.endX - focusAnnotation.x) + 10 + 'px',
            height: Math.abs(focusAnnotation.endY - focusAnnotation.y) + 10 + 'px',
          }"
        ></section>
      </a-col>
      <a-col :span="5" class="detail-box">
        <div class="head-box">共 {{ annotationsList.length }} 条批注</div>
        <a-divider style="margin: 10px 0" />
        <div class="card-box">
          <a-card
            hoverable
            size="small"
            v-for="(item, i) in annotationsList"
            :key="i"
            :class="`${item.editStatus && 'current'} annotation-card`"
            @click="mouseClickAnnotation(i)"
            @mouseleave="mouseLeaveAnnotation"
          >
            <template slot="actions" v-if="annotationMode === 'Scrawl'">
              <a v-if="!item.editStatus" @click="item.editStatus = true"><a-icon type="edit" /></a>
              <a v-else @click="saveAnnotation(i)"><a-icon type="save" /></a>
              <a style="margin-left: 10px" @click="removeAnnotation(i)"><a-icon type="delete" /></a>
            </template>
            <a-card-meta :description="item.createTime" style="margin-bottom: 10px">
              <a-avatar slot="avatar" src="https://zos.alipayobjects.com/rmsportal/ODTLcjxAfvqbxHnVXCYX.png" />
            </a-card-meta>
            <span v-if="!item.editStatus">{{ item.comments }}</span>
            <a-textarea
              v-else
              v-model="item.comments"
              placeholder="请输入内容"
              :auto-size="{ minRows: 1, maxRows: 2 }"
              @change="(e) => textareaChange(e, item)"
            />
            <div class="dropdown" v-show="item.showDropdown">
              <div class="dropdown-item" v-for="(option, index) in manData" :key="index" @click="chooseMan(index, i)">
                {{ option.realname }}
              </div>
            </div>
          </a-card>
        </div>
      </a-col>
    </a-row>
  </div>
</template>

<script>
import jsPDF from 'jspdf'
import * as pdfjs from 'pdfjs-dist'
import pdfjsWorker from 'pdfjs-dist/build/pdf.worker.entry'
pdfjs.GlobalWorkerOptions.workerSrc = pdfjsWorker

import guaz from '@/assets/guaz.png'

export default {
  name: 'PdfViewer',
  props: {
    file: {
      type: Object | String,
    },
    annotationData: {
      type: Array,
      require: true,
      default: () => [],
    },
    manData: {
      type: Array,
      require: true,
      default: () => [],
    },
    readonly: {
      type: Boolean,
      require: false,
      default: false,
    },
  },
  data() {
    return {
      toolItems: [
        {
          icon: 'eye',
          type: 'Empty',
          checked: true,
        },
        {
          icon: 'edit',
          type: 'Scrawl',
          checked: false,
          sub: [
            // {
            //   icon: 'font-size',
            //   type: 'Text',
            //   checked: false,
            //   text: '文本',
            // },
            {
              icon: 'message',
              type: 'Mark',
              checked: false,
              text: '标记点',
            },
            {
              icon: 'edit',
              type: 'Line',
              checked: false,
              text: '线',
            },
            {
              icon: 'line',
              type: 'StraightLine',
              checked: false,
              text: '直线',
            },
            {
              icon: 'transaction',
              type: 'WaveLine',
              checked: false,
              text: '波浪线',
            },
            {
              icon: 'border',
              type: 'Rectangle',
              checked: false,
              text: '矩形',
            },
            {
              icon: 'radius-upright',
              type: 'Elliptic',
              checked: false,
              text: '椭圆',
            },
          ],
        },
        {
          icon: 'download',
          type: 'Download',
          checked: false,
        },
      ],
      pdfjs: '',
      scale: 2,
      last_scale: 2,
      resource: this.file,
      annotationsList: JSON.parse(JSON.stringify(this.annotationData)),
      currentPage: 0,
      annotationMode: 'Empty', // tool flag
      annotation: {
        lineWidth: 1,
        fontSize: 16,
        color: '',
        scrawls: [],
      }, // the object for create new annotation or scrawls
      inputTextValueVisible: false,
      canvasList: [],
      pdfContainer: null,
      scrollContainer: null,
      isDrawing: false, // the control flag when line/reactangle and so on.
      scrawlType: '', // the flag about sub menu tools
      pagination: {
        current: 1,
        total: 0,
      },
      zoomNumber: 200,
      focusAnnotation: {},
    }
  },
  watch: {
    annotationData(val) {
      this.annotationsList = JSON.parse(JSON.stringify(val))
      this.zoomChange(this.zoomNumber)
    },
    file(val) {
      this.resource = val
      this.init()
    },
  },
  mounted() {
    this.init()
  },
  methods: {
    init() {
      this.pdfContainer = document.getElementById('pdfContainer')
      this.scrollContainer = document.querySelector('.pdf-box')
      pdfjs.getDocument(this.resource).promise.then((res) => {
        this.pdfjs = res
        this.zoomChange(this.zoomNumber)
      })
    },
    async renderAllPages() {
      this.$set(this.pagination, 'total', this.pdfjs.numPages)
      this.pdfContainer.innerHTML = null
      this.canvasList = []
      for (let pageNum = 1; pageNum <= this.pdfjs.numPages; pageNum++) {
        const page = await this.pdfjs.getPage(pageNum)
        const viewport = page.getViewport({ scale: this.scale })
        const canvas = document.createElement('canvas')
        canvas.width = viewport.width
        canvas.height = viewport.height
        canvas.className = 'canvasCanvas'
        canvas.style.display = 'block'
        const context = canvas.getContext('2d')
        context.imageSmoothingEnabled = true
        this.pdfContainer.appendChild(canvas)
        this.canvasList.push(canvas)

        this.updatePage(page, context, viewport).then(() => {
          this.annotationsList.forEach((annotation) => {
            if (annotation.page === pageNum) {
              this.initAllKindsAnnotation(context, annotation)
            }
          })
        })
      }
      return new Promise((resolve) => {
        resolve(this.canvasList)
      })
    },
    async updatePage(page, context, viewport) {
      return new Promise((resolve) => {
        page
          .render({
            canvasContext: context,
            viewport: viewport,
          })
          .promise.then(() => {
            resolve()
          })
      })
    },
    layerClick(e, pageNum, viewport, scrawlType) {
      const x = e.offsetX
      const y = e.offsetY
      this.annotation = {
        ...this.annotation,
        x,
        y,
        yEditor: pageNum === 1 ? y : Math.round(viewport.height) * (pageNum - 1) + 5.5 * (pageNum - 1) + y,
        width: '',
        height: '',
        textValue: '',
        comments: '',
        type: 'Scrawl',
        scrawlType: scrawlType,
        page: pageNum,
        editStatus: true,
        scrawls: [],
        absoluteY: pageNum === 1 ? y : Math.round(viewport.height) * (pageNum - 1) + 5.5 * (pageNum - 1) + y,
        endX: x + 30,
        endY: y + 30,
      }
    },
    addAnnotation(annotation) {
      if (!annotation.textValue) return
      const canvas = this.canvasList[annotation.page - 1]
      const context = canvas.getContext('2d')
      const textWidth = context.measureText(annotation.textValue).width
      const textHeight = parseFloat(annotation.fontSize)
      annotation.width = textWidth
      annotation.height = textHeight
      annotation.editStatus = true
      annotation.type = 'Scrawl'
      annotation.scrawlType = 'Text'
      this.annotationsList.push(annotation)
      this.annotation = { ...this.annotation, x: '', textValue: '' }
      this.inputTextValueVisible = false
      this.drawText(context, annotation)
    },
    editAnnotation(index) {
      this.annotationsList.forEach((v) => {
        v.editStatus = false
      })
      const item = this.annotationsList[index]
      item.editStatus = true
      this.annotation = item
      this.$set(this.annotationsList, index, item)
    },
    scaleAnnotationList() {
      this.annotationsList.forEach((annotation) => {
        annotation.editStatus = false
        annotation.showDropdown = false
        annotation.width = +((annotation.width / this.last_scale) * this.scale).toFixed(4)
        annotation.height = +((annotation.height / this.last_scale) * this.scale).toFixed(4)
        annotation.x = +((annotation.x / this.last_scale) * this.scale).toFixed(4)
        annotation.y = +((annotation.y / this.last_scale) * this.scale).toFixed(4)
        annotation.yEditor = +((annotation.yEditor / this.last_scale) * this.scale).toFixed(4)
        annotation.endX = +((annotation.endX / this.last_scale) * this.scale).toFixed(4)
        annotation.endY = +((annotation.endY / this.last_scale) * this.scale).toFixed(4)
        annotation.absoluteEndY = +((annotation.absoluteEndY / this.last_scale) * this.scale).toFixed(4)
        annotation.absoluteY = +((annotation.absoluteY / this.last_scale) * this.scale).toFixed(4)
        annotation.absoluteEndY = +((annotation.absoluteEndY / this.last_scale) * this.scale).toFixed(4)
        annotation.lineWidth = +((annotation.lineWidth / this.last_scale) * this.scale).toFixed(4)
        annotation.fontSize = +((annotation.fontSize / this.last_scale) * this.scale).toFixed(4)
        if (annotation.hasOwnProperty('scrawls')) {
          annotation.scrawls.forEach((line) => {
            line.x = +((line.x / this.last_scale) * this.scale).toFixed(4)
            line.y = +((line.y / this.last_scale) * this.scale).toFixed(4)
          })
        }
      })
    },
    updateStorage(handler, data) {
      this.$emit('listenerForDataChange', handler, data, this.annotationsList)
    },
    drawText(context, annotation) {
      context.fillStyle = annotation.color
      context.font = `${annotation.fontSize || 12}px Arial`
      context.fillText(annotation.textValue, annotation.x, annotation.y)
    },
    async saveAndDownloadAnnotatedPDF() {
      const jspdf = new jsPDF('a4')
      jspdf.setFont('SIMHEI')
      for (let i = 0; i < this.canvasList.length; i++) {
        if (i > 0) {
          jspdf.addPage()
        }
        const canvas = this.canvasList[i]
        const annoCvs = canvas.toDataURL('image/jpeg', 1.0)
        jspdf.addImage(annoCvs, 'JPEG', 0, 0, 210, 297)
      }
      jspdf.save('canvas_to_pdf.pdf')
    },
    getPdfStream() {
      const jspdf = new jsPDF('a4')
      jspdf.setFont('SIMHEI')
      for (let i = 0; i < this.canvasList.length; i++) {
        if (i > 0) {
          jspdf.addPage()
        }
        const canvas = this.canvasList[i]
        const annoCvs = canvas.toDataURL('image/jpeg', 1.0)
        jspdf.addImage(annoCvs, 'JPEG', 0, 0, 210, 297)
      }
      return new Promise((resolve) => {
        resolve(jspdf.output('blob'))
      })
    },
    async changeAnnotationType(item) {
      this.toolItems.forEach((v) => {
        this.$set(v, 'popOpen', false)
      })
      this.$set(item, 'popOpen', true)
      if (item.checked) return
      this.annotationMode = item.type
      switch (this.annotationMode) {
        case 'Empty':
          break
        // case 'Scrawl':
        //   this.scrawlModePrepare()
        //   break
        case 'Download':
          this.saveAndDownloadAnnotatedPDF()
          break

        default:
          break
      }
      this.toolItems.forEach((v) => {
        v.checked = false
      })
      this.$set(item, 'checked', true)
    },
    scrawlMouseDown(e, i, canvas) {
      if (this.annotationMode !== 'Scrawl') return
      if (this.scrawlType === 'Text') return
      this.isDrawing = true
      this.annotation = {
        x: e.offsetX,
        y: e.offsetY,
        type: 'Scrawl',
        color: this.annotation.color,
        lineWidth: this.annotation.lineWidth,
        scrawlType: this.scrawlType,
        page: i + 1,
        absoluteY: `${i === 0 ? e.offsetY : Math.round(canvas.height) * i + 5.5 * i + e.offsetY}`,
      }
      if (this.scrawlType === 'Line') {
        this.annotation.scrawls = [{ x: e.offsetX, y: e.offsetY }]
      }
    },
    scrawlMouseMove(e, context, i, canvas) {
      if (this.annotationMode !== 'Scrawl') return
      if (!this.isDrawing) return
      const x = e.offsetX
      const y = e.offsetY
      if (this.scrawlType === 'Line') {
        this.annotation.scrawls.push({ x, y })
        this.drawLine(context, x, y)
      } else if (
        this.scrawlType === 'Rectangle' ||
        this.scrawlType === 'Elliptic' ||
        this.scrawlType === 'StraightLine' ||
        this.scrawlType === 'WaveLine'
      ) {
        this.$set(this.annotation, 'endX', x)
        this.$set(this.annotation, 'endY', y)
        this.$set(this.annotation, 'absoluteEndY', `${i === 0 ? y : Math.round(canvas.height) * i + 5.5 * i + y}`)
      }
    },
    scrawlMouseUp(context) {
      if (this.annotationMode !== 'Scrawl') return
      if (!this.scrawlType || this.scrawlType === 'Text' || this.scrawlType === 'Mark') return
      this.annotation.editStatus = true
      this.annotation.checked = true
      switch (this.scrawlType) {
        case 'Rectangle':
          this.drawRectangle(context, this.annotation)
          break
        case 'Elliptic':
          this.drawEllipse(context, this.annotation)
          break
        case 'StraightLine':
          this.drawStraightLine(context, this.annotation)
          break
        case 'WaveLine':
          this.drawWaveLine(context, this.annotation)
          break

        default:
          break
      }
      this.annotationsList.push(this.annotation)
      this.isDrawing = false
    },
    scrawlMouseClick(e, i, canvas, context) {
      if (this.annotationMode !== 'Scrawl') return
      this.isDrawing = false
      this.toolItems.forEach((v) => {
        v.popOpen = false
      })
      const pageNum = i + 1
      this.layerClick(e, pageNum, canvas, this.scrawlType)
      if (this.scrawlType === 'Text') {
        this.inputTextValueVisible = true
      } else if (this.scrawlType === 'Mark') {
        this.drawMark(context, this.annotation)
        this.annotationsList.push(this.annotation)
      }
    },
    scrawlModePrepare() {
      this.canvasList.forEach((canvas, i) => {
        const context = canvas.getContext('2d')
        canvas.addEventListener('mousedown', (e) => this.scrawlMouseDown(e, i, canvas))
        canvas.addEventListener('mousemove', (e) => this.scrawlMouseMove(e, context, i, canvas))
        canvas.addEventListener('mouseup', () => this.scrawlMouseUp(context))
        canvas.addEventListener('click', (e) => this.scrawlMouseClick(e, i, canvas, context))
      })
    },
    drawWaveLine(context, annotation) {
      context.strokeStyle = annotation.color
      context.lineWidth = annotation.lineWidth
      const distance = Math.sqrt((annotation.endX - annotation.x) ** 2 + (annotation.endY - annotation.y) ** 2)
      const frequency = 0.1
      const amplitude = 10
      const numWaves = 5
      context.beginPath()
      context.moveTo(annotation.x, annotation.y)
      for (let i = 0; i <= distance; i += 5) {
        const x = annotation.x + i
        const y = annotation.y + Math.sin(x * frequency) * amplitude * Math.sin((i / distance) * Math.PI * numWaves)
        context.lineTo(x, y)
      }
      context.stroke()
    },
    drawStraightLine(context, annotation) {
      context.strokeStyle = annotation.color
      context.lineWidth = annotation.lineWidth
      context.beginPath()
      context.moveTo(annotation.x, annotation.y)
      context.lineTo(annotation.endX, annotation.endY)
      context.stroke()
    },
    drawMark(context, annotation) {
      const icon = new Image()
      icon.src = guaz
      icon.onload = () => {
        context.drawImage(icon, annotation.x, annotation.y, 30, 30)
      }
    },
    drawLine(context, x, y) {
      context.strokeStyle = this.annotation.color
      context.lineWidth = this.annotation.lineWidth
      context.beginPath()
      context.moveTo(this.annotation.x, this.annotation.y)
      context.lineTo(x, y)
      context.stroke()
      this.annotation = {
        ...this.annotation,
        x,
        y,
      }
    },
    initLines(context, annotation) {
      context.beginPath()
      context.strokeStyle = annotation.color
      context.lineWidth = annotation.lineWidth
      context.moveTo(annotation.scrawls[0].x, annotation.scrawls[0].y)
      annotation.scrawls.forEach((point) => {
        context.lineTo(point.x, point.y)
      })
      context.stroke()
    },
    scrawlClick(i, k) {
      this.toolItems.forEach((v) => {
        v.checked = false
        if (v.hasOwnProperty('sub')) {
          v.sub.forEach((o) => {
            o.checked = false
          })
        }
      })
      this.toolItems[i].checked = true
      this.toolItems[i].sub[k].checked = true
      const sub = this.toolItems[i].sub
      this.scrawlType = sub[k].type
      this.$set(this.toolItems[i], 'sub', sub)
      this.zoomChange(this.zoomNumber)
    },
    drawRectangle(context, annotation) {
      const { x, y, endX, endY } = annotation
      context.strokeStyle = annotation.color
      context.lineWidth = annotation.lineWidth
      context.strokeRect(x, y, endX - x, endY - y)
    },
    initAllKindsAnnotation(context, annotation) {
      if (annotation.type === 'Scrawl') {
        switch (annotation.scrawlType) {
          case 'Text':
            this.drawText(context, annotation)
            break
          case 'Mark':
            this.drawMark(context, annotation)
            break
          case 'Line':
            this.initLines(context, annotation)
            break
          case 'StraightLine':
            this.drawStraightLine(context, annotation)
            break
          case 'WaveLine':
            this.drawWaveLine(context, annotation)
            break
          case 'Rectangle':
            this.drawRectangle(context, annotation)
            break
          case 'Elliptic':
            this.drawEllipse(context, annotation)
            break

          default:
            break
        }
      }
    },
    drawEllipse(context, annotation) {
      const { x, y, endX, endY } = annotation
      const centerX = (x + endX) / 2
      const centerY = (y + endY) / 2
      const radiusX = Math.abs(endX - x) / 2
      const radiusY = Math.abs(endY - y) / 2
      context.beginPath()
      context.strokeStyle = annotation.color
      context.lineWidth = annotation.lineWidth
      context.ellipse(centerX, centerY, radiusX, radiusY, 0, 0, 2 * Math.PI)
      context.closePath()
      context.stroke()
    },
    closePop(index) {
      const item = this.toolItems[index]
      item.popOpen = false
      this.$set(this.toolItems, index, item)
    },
    pageLocated(page) {
      const targetPage = this.canvasList[page - 1]
      if (targetPage) {
        this.scrollContainer.scrollTo({
          top: targetPage.offsetTop,
          behavior: 'smooth',
        })
      }
    },
    locatedWhenScroll() {
      this.scrollContainer.addEventListener('scroll', () => {
        const scrollPosition = this.scrollContainer.scrollTop
        this.canvasList.forEach((page, index) => {
          const pageTop = page.offsetTop
          const pageBottom = pageTop + page.clientHeight
          if (scrollPosition >= pageTop && scrollPosition < pageBottom) {
            const currentPageNumber = index + 1
            this.$set(this.pagination, 'current', currentPageNumber)
          }
        })
      })
    },
    zoomIn() {
      this.zoomNumber += 10
      this.zoomChange(this.zoomNumber)
    },
    zoomOut() {
      this.zoomNumber -= 10
      this.zoomChange(this.zoomNumber)
    },
    zoomChange(val) {
      this.zoomNumber = val
      this.last_scale = this.scale
      this.scale = this.zoomNumber / 100
      this.scaleAnnotationList()
      this.renderAllPages().then((res) => {
        this.scrawlModePrepare()
        this.locatedWhenScroll()
        this.pageLocated(this.pagination.current)
      })
    },
    enterKey(event) {
      event = event || window.event
      if (event.keyCode == 13) {
        this.addAnnotation(this.annotation)
      }
    },
    removeAnnotation(index) {
      const item = this.annotationsList[index]
      this.annotationsList.splice(index, 1)
      this.annotation.textValue = ''
      this.annotation.x = ''
      this.updateStorage('delete', item)
    },
    saveAnnotation(index) {
      const item = this.annotationsList[index]
      item.editStatus = false
      this.$set(this.annotationsList, index, item)
      this.updateStorage('addOrEdit', item)
    },
    textareaChange(e, item) {
      const text = e.target.value
      const lastChar = text[text.length - 1]
      if (lastChar === '@') {
        item.showDropdown = true
      } else {
        item.showDropdown = false
      }
    },
    chooseMan(index, i) {
      const item = this.annotationsList[i]
      item.username = this.manData[index].username
      item.realname = this.manData[index].realname
      item.comments = item.comments + item.realname
      item.showDropdown = false
      this.$set(this.annotationsList, i, item)
    },
    mouseClickAnnotation(index) {
      const item = this.annotationsList[index]
      const i = item.page - 1
      if (item.scrawlType === 'Line') {
        const result = this.calculateHandler(item.scrawls)
        this.focusAnnotation = {
          x: result.minX,
          y: result.minY,
          absoluteY: `${i === 0 ? result.minY : Math.round(this.canvasList[0].height) * i + 5.5 * i + result.minY}`,
          endX: result.maxX,
          endY: result.maxY,
        }
      } else {
        this.focusAnnotation = this.annotationsList[index]
      }
      this.pageLocated(item.page)
    },
    mouseLeaveAnnotation() {
      this.focusAnnotation = {}
    },
    calculateHandler(coordinates) {
      const result = coordinates.reduce(
        (acc, coord) => {
          acc.minX = Math.min(acc.minX, coord.x)
          acc.minY = Math.min(acc.minY, coord.y)
          acc.maxX = Math.max(acc.maxX, coord.x)
          acc.maxY = Math.max(acc.maxY, coord.y)
          return acc
        },
        {
          minX: Infinity,
          minY: Infinity,
          maxX: -Infinity,
          maxY: -Infinity,
        }
      )
      return result
    },
  },
}
</script>

<style lang="less">
.font-size-color-wrapper {
  .font-size-color-box {
    display: flex;
    justify-content: space-between;
    align-content: center;
    margin-bottom: 10px;

    .label-box {
      display: flex;
      align-items: center;
    }
  }
}
.sub-box {
  padding: 5px;
  margin-bottom: 10px;
  &:hover {
    background-color: silver;
  }
  i {
    font-size: 15px;
    vertical-align: text-bottom;
    margin-right: 10px;
  }
  &.checked {
    background-color: green;
    color: #fff;
  }
}
.tool-wrapper {
  background-color: #fff;
  border-radius: 5px;
  margin-bottom: 5px;
  display: flex;
  justify-content: space-between;
  align-content: center;
  .tool-box {
    i {
      padding: 5px;
      margin: 10px;
      border-radius: 5px;
      &.checked {
        background-color: green;
        color: #fff;
      }
    }
  }
  .pagination-box {
    display: flex;
    align-items: center;
    .ant-pagination-item-link {
      margin-top: 0;
      margin-bottom: 0;
    }
  }

  .zoom-box {
    width: 160px;
    display: flex;
    justify-content: space-between;
    align-items: center;
    i {
      margin: 0 10px;
    }
  }
}

.detail-box {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  height: 550px;
  padding: 0 5px 10px 5px;
  background-color: #ededf0;
  border: 1px solid #e8e8e8;
  border-top: none;
  border-radius: 3px;
  .head-box {
    display: flex;
    justify-content: space-between;
    align-items: center;
    background-color: #fff;
    padding: 5px;
    border-radius: 3px;
  }

  .annotation-card {
    margin-bottom: 10px;
    &:hover {
      background-color: yellow;
    }
  }
}

.popover-box {
  font-size: 11px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 200px;
  margin-bottom: 10px;
  i {
    font-size: 20px;
  }
}

.pdf-wrapper {
  .pdf-box {
    height: 550px;
    overflow: auto;
    .editor-input {
      position: absolute;
      width: 200px;
      background-color: transparent !important;
      border: 1px solid #2470bd !important;
    }
    .editor-section {
      position: absolute;
      .input-delete {
        display: flex;
        align-items: center;
        .ant-input {
          width: 200px;
          background-color: transparent !important;
          border: 1px solid #2470bd !important;
        }
        i {
          margin-left: 10px;
        }
      }
    }
    .focus-section {
      position: absolute;
      border: 2px dashed blue;
    }
  }
}

.card-box {
  flex: 1;
  overflow: auto;
  padding: 0;
  .ant-card {
    width: calc(100% - 10px);
    &.current {
      background-color: yellow;
    }
  }
  .dropdown {
    position: absolute;
    border: 1px solid #ddd;
    max-height: 100px;
    overflow-y: auto;
    background-color: #fff;
    width: calc(90% - 5px);
    z-index: 10;
    padding: 5px 0;
    .dropdown-item {
      padding: 5px;
      &:hover {
        background-color: #ddd;
      }
    }
  }
}
</style>