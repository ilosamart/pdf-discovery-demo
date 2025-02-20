<template>
  <div class="the-pdf-viewer" id="the-frb-pdf-viewer">
    <div class="pdf-viewer-container" ref="pdf-viewer-container">
      <div id="pdf-viewer" class="pdf-viewer" ref="pdf-viewer" />
    </div>
  </div>
</template>

<script>
import * as PDFJS from 'pdfjs-dist/web/pdf_viewer'
import 'pdfjs-dist/web/pdf_viewer.css'

export default {
  props: {
    maxWidth: {
      type: Number,
      default: 900
    },
    highlights: {
      type: Object,
      default: () => {}
    },
    id: {
      type: String,
      required: true
    },
    documentPath: {
      type: String,
      default: '/'
    }
  },
  data () {
    return {
      pdfjsLib: null,
      PDFViewer: null,
      viewportWidth: 0
    }
  },
  mounted () {
    this.pdfjsLib = require('pdfjs-dist/webpack')
    this.initializeViewer()
  },
  computed: {
    pdfScale () {
      return this.viewportWidth >= this.maxWidth ? 1 : 'page-width'
    }
  },
  methods: {
    triggerResize () {
      this.viewportWidth = this.$refs['pdf-viewer-container'].offsetWidth
      this.PDFViewer.currentScaleValue = 1.0001
      this.PDFViewer.currentScaleValue = this.pdfScale
    },
    initializeViewer () {
      this.PDFViewer = new PDFJS.PDFViewer({
        container: this.$refs['pdf-viewer-container'],
        linkService: this.PDFLinkService
      })
      document.addEventListener('pagesinit', () => {
        this.triggerResize()
      })
      window.addEventListener('resize', () => {
        this.triggerResize()
      })
      document.addEventListener('pagerendered', (e) => {
        this.renderHighlightsOnPage(e.detail.pageNumber)
      })

      let loadingTask = this.pdfjsLib.getDocument({
        // url: `http://${window.location.hostname}:8443/${this.id}`
        url: `${this.documentPath}${this.id}`
      })
      loadingTask.promise.then((pdfDocument) => {
        this.PDFViewer.setDocument(pdfDocument)
      })
    },
    renderHighlightsOnPage (pageNumber) {
      if (this.highlights && this.highlights.payloads) {
        Object.keys(this.highlights.payloads).forEach(payloadDoc => {
          let highlightTerms = this.highlights.payloads[payloadDoc].SpeechContentOcr
            ? this.highlights.payloads[payloadDoc].SpeechContentOcr
            : this.highlights.payloads[payloadDoc].content_ocr
          Object.keys(highlightTerms).forEach(term => {
            highlightTerms[term].forEach(highlight => {
              let [targetPageNumber, ...coordinates] = highlight.payload
                ? highlight.payload.split(' ').filter(Number)
                : highlight.split(' ').filter(Number)
              if (pageNumber.toString() === targetPageNumber.toString()) {
                this.addHighlightToPDF(pageNumber, coordinates, highlight.startOffset, highlight.endOffset)
              }
            })
          })
        })
      } else {
        console.error('target PDF not found in result set')
      }
    },
    addHighlightToPDF (pageNumber, coordinates, startOffset, endOffset) {
      let doc = this.highlights.pageDict[pageNumber]
      let pageDimension = doc.PageDimension
        ? doc.PageDimension
        : doc.page_dimension
      let [pageWidth, pageHeight] = pageDimension[0].trim().split(' ').slice(2)
      let [x1, y1, x2, y2] = coordinates
      let highlight = document.createElement('span')
      let targetPage = document.querySelector(`[data-page-number="${pageNumber}"]`)

      // set up highlight position and size
      const highlightSize = {}
      highlightSize.top = (y1 / pageHeight) * 100
      highlightSize.left = (x1 / pageWidth) * 100
      highlightSize.height = ((y2 / pageHeight) - (y1 / pageHeight)) * 100
      highlightSize.width = ((x2 / pageWidth) - (x1 / pageWidth)) * 100

      Object.keys(highlightSize).forEach((dimension) => {
        if (!highlightSize[dimension]) {
          console.error(`Undefined sizing variable for highlight: highlightSize.${dimension} evaluated to ${highlightSize[dimension]}. Highlight will not render correctly. This is probably an issue with payload data being provided to the application.`)
        }
      })

      // set the relative position for the highlight
      highlight.setAttribute('data-end-offset', endOffset)
      highlight.setAttribute('data-start-offset', startOffset)
      highlight.setAttribute('class', 'box-highlight')
      highlight.setAttribute('style', `
        top:${highlightSize.top}%;
        left:${highlightSize.left}%;
        height:${highlightSize.height}%;
        width:${highlightSize.width}%;
      `)

      // add the highlight to the page
      targetPage.appendChild(highlight)
    }
  }
}
</script>

<style lang="scss" scoped>
.pdf-viewer-container {
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
  position: absolute;
  overflow-y: scroll;
}

.pdf-viewer {
  display: flex;
  flex-direction: column;
  align-items: center;

  & /deep/ .page {
    margin-left: auto;
    margin-right: auto;
    margin-bottom: 4em;
    position: relative;
    max-width: 100%;
    box-shadow: 2px 0 1.5em 0 rgba(#2f3235, 0.4);

    ::-moz-selection {
      background: blue;
      color: transparent;
    }
    ::selection {
      background: blue;
      color: transparent;
    }

    &:first-child {
      margin-top: 12em;
    }

    &:last-child {
      margin-bottom: 4em;
    }

    .box-highlight {
      position: absolute;
      display: block;
      background: blue;
      opacity: 0.2;
      transform-origin: center center;
      transform: scale(1.1, 1.15);
      border-radius: 2px;
      pointer-events: none;
    }
  }
}
</style>
