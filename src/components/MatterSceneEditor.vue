<template>
    <div>
        <div>
            <svg width="800" height="800" id="base"></svg>
        </div>
        <div style="margin-top: 20px;">
            <button @click="setElementType('wall')">Wall (Lines)</button>
            <button @click="setElementType('wall-sphere')">Wall (Sphere)</button>
            <button @click="setElementType('gate')">Gate</button>
            <button @click="setElementType('windstream')">Windstream</button>
            <button @click="setElementType('ball')">Ball</button>
        </div>
        <div>
            <input type="text" v-model="gridSubdiv"/>
            <button @click="updateDotGrid">Update Dot Grid</button>
        </div>
        <div>
            <label>Fixed Circle Size: </label>
            <input type="text" @input="updateFixedCircleSize" v-model="fixedCircleSize" />
        </div>
        <div v-if="selectedPolygon" style="padding: 5px; margin-top: 5px; border: lightgray solid 1px;">
            <div v-for="key in attrs" v-bind:key="key">
                <label :for="key">{{ key }}</label>
                <input type="text" @input="updateSvgAttribute(selectedPolygon, key, $event)" :value="getSvgAttribute(selectedPolygon, key)" />
            </div>
        </div>
        <div style="margin-top: 5px;">
            <button @click="generateOutputJson">Generate</button>
            <button @click="showDialog">Import</button>
        </div>
        <dialog ref="inputDialog">
            <div>
                <textarea v-model="importShapeText" rows="20" cols="80" />
            </div>
            <button @click="importShapes">Import</button>
            <button @click="hideDialog">Close</button>
        </dialog>
    </div>
</template>

<script>
import { Canvas, DrawHandler, DrawPolygonHandler, DrawCircleHandler } from '@/editor.js'

let drawHandler = null

export default {
    name: 'MatterSceneEditor',
    data() {
        return {
            importShapeText: '',
            selectedPolygon: null,
            shiftHeld: false,
            controlHeld: false,
            finalShapes: [],
            svg: null,
            gridSubdiv: 20,
            fixedCircleSize: null
        }
    },
    computed: {
        attrs() {
            return this.selectedPolygon.getAttributeNames().filter(key => key !== 'stroke')
        }
    },
    mounted() {
        this.svg = document.getElementById('base')
        let canvas = new Canvas(this.svg, this.gridSubdiv)
        drawHandler = new DrawHandler(canvas, this.peristShapeHandler)
        // start with wall element types by default
        this.setElementType('wall')

        document.addEventListener('keydown', (event) => {
            if(event.key == "Escape") {
                if(this.selectedPolygon) {
                    // polygon current selected, to deselect it
                    this.selectPolygon(this.selectedPolygon, false)
                    this.selectedPolygon = false
                }
                // cancel current drawing action (default)
                 drawHandler.cancelDrawing()
            }
            else if(event.key == 'Delete') {
                this.deleteSelectedPolygon()
            }
            else if(event.key == 'Shift') this.shiftHeld = true
            else if(event.key == 'Control') this.controlHeld = true
            else if(event.key == 'z') {
                if(this.controlHeld && this.finalShapes.length > 0) {
                    // undo last shape added
                    const lastPolygonCreated = this.finalShapes.pop()
                    this.svg.removeChild(lastPolygonCreated)
                    this.selectedPolygon = null
                }
            }
        })

        document.addEventListener('keyup', (event) => {
            if(event.key == 'Shift') this.shiftHeld = false
            else if(event.key == 'Control') this.controlHeld = false
        })
    },
    methods: {
            peristShapeHandler(svgShape) {
                this.finalShapes.push(svgShape)
                const self = this
                svgShape.addEventListener('click', function(event) {
                    self.selectedPolygon = svgShape
                    self.shapeClickHandler(event)
                })
            },
            showDialog() {
                this.$refs.inputDialog.showModal()
            },
            importShapes() {
                if(!this.importShapeText || this.importShapeText.trim().length == 0) return

                // remove all existing shapes
                for(const shape of this.finalShapes) this.svg.removeChild(shape)
                this.finalShapes = []

                const canvas = drawHandler.getCanvas()
                const shapeJson = JSON.parse(this.importShapeText)
                for(const shapeData of shapeJson) {
                    const { svgElementType, ...attrs } = shapeData
                    const shape = canvas.createSvgElement(svgElementType, attrs)
                    canvas.addSvgElement(shape)
                    this.peristShapeHandler(shape)
                }

                this.hideDialog()
            },
            hideDialog() {
                this.$refs.inputDialog.close()
            },
            updateFixedCircleSize() {
                if(this.fixedCircleSize.trim().length == 0) this.fixedCircleSize = null
                else this.fixedCircleSize = parseInt(this.fixedCircleSize)
            },
            generateOutputJson() {
                console.log(this.finalShapes.map(svgShape => {
                    return svgShape.getAttributeNames()
                        .filter(key => key !== 'stroke')
                        .reduce((acc, curr) => {
                            return {
                                ...acc,
                                [curr]: svgShape.getAttribute(curr)
                            }
                        }, {
                            svgElementType: svgShape.tagName
                        })
                }))
            },
            getSvgAttribute(element, attributeName) {
                return element.getAttribute(attributeName)
            },
            updateSvgAttribute(element, attributeName, event) {
                element.setAttribute(attributeName, event.target.value)
            },
            updateSvgAttributes(element, attributes) {
                for(const key in attributes) element.setAttribute(key, attributes[key])
            },
            updateDotGrid() {
                const canvas = drawHandler.getCanvas()
                canvas.createDotGrid(canvas.getSvg(), parseInt(this.gridSubdiv))
            },
            shapeClickHandler(event) {
                // ignore shape clicks if shift is held
                if(this.shiftHeld) return

                // if a polygon is already selected, deselect the previous one
                if(this.selectedPolygon) this.selectPolygon(this.selectedPolygon, false)
                this.selectedPolygon = event.srcElement
                this.selectPolygon(this.selectedPolygon, true)
                event.stopPropagation()
            },
            selectPolygon(polygon, selected) {
                this.updateSvgAttributes(polygon, {
                    stroke: selected ? 'red' : null
                })
            },
            deleteSelectedPolygon() {
                if(this.selectedPolygon) {
                    this.svg.removeChild(this.selectedPolygon)
                    this.finalShapes = this.finalShapes.filter((shape) => shape.id != this.selectedPolygon.id)
                    this.selectedPolygon = null
                }
            },
            setElementType(type) {
                let attributes = {
                    elementType: type,
                    fill: '#000',
                    opacity: 0.3
                }

                if(type == 'wall') attributes.fill = '#000000'
                if(type == 'wall-sphere') attributes.fill = '#000000'
                if(type == 'gate') {
                    attributes.fill = '#0000FF'
                }
                if(type == 'windstream') {
                    attributes.fill = '#A020F0'
                    attributes.xForce = 0.00006
                    attributes.yForce = 0
                }
                if(type == 'ball') attributes.fill = '#FF0000'

                let drawShapeHandler = null
                if(['wall-sphere','ball'].indexOf(type) != -1) {
                    if(this.fixedCircleSize) {
                        attributes = {
                            ...attributes,
                            r: this.fixedCircleSize
                        }
                    }
                    drawShapeHandler = new DrawCircleHandler(drawHandler, attributes)
                } else {
                    drawShapeHandler = new DrawPolygonHandler(drawHandler, attributes)
                }
                drawHandler.setShapeHandler(drawShapeHandler)
            }
    }
}
</script>

<style>

</style>