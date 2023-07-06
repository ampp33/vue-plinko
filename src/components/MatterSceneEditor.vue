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
        <div v-if="selectedPolygon" style="padding: 5px; margin-top: 5px; border: lightgray solid 1px;">
            <div v-for="key in attrs" v-bind:key="key">
                <label :for="key">{{ key }}</label>
                <input type="text" @input="updateSvgAttribute(selectedPolygon, key, $event)" :value="getSvgAttribute(selectedPolygon, key)" />
            </div>
        </div>
        <div style="margin-top: 5px;">
            <button @click="generateOutputJson">Generate</button>
        </div>
    </div>
</template>

<script>
import { Canvas, DrawHandler, DrawPolygonHandler, DrawCircleHandler } from '@/editor.js'

let drawHandler = null

export default {
    name: 'MatterSceneEditor',
    data() {
        return {
            selectedPolygon: null,
            ignorePolygonSelection: false,
            finalShapes: [],
            svg: null,
            gridSubdiv: 20
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
        drawHandler = new DrawHandler(canvas, (svgShape) => {
            this.finalShapes.push(svgShape)
            const self = this
            svgShape.addEventListener('click', function(event) {
                self.selectedPolygon = svgShape
                self.shapeClickHandler(event)
            })
        })
        // start with wall element types by default
        this.setElementType('wall')

        document.addEventListener('keydown', (event) => {
            if(event.key == "Escape") {
                if(this.selectedPolygon) {
                    // polygon current selected, to deselect it
                    this.selectPolygon(this.selectedPolygon, false)
                    this.selectedPolygon = false
                    return
                }
                // cancel current drawing action (default)
                 drawHandler.cancelDrawing()
            }
            else if(event.key == 'Delete') {
                this.deleteSelectedPolygon()
            }
            else if(event.key == 'Shift') {
                this.ignorePolygonSelection = true
            }
        })

        document.addEventListener('keyup', (event) => {
            if(event.key == 'Shift') {
                this.ignorePolygonSelection = false
            }
        })
    },
    methods: {
            generateOutputJson() {
                console.log(this.finalShapes.map(svgShape => {
                    return svgShape.getAttributeNames()
                        .filter(key => key !== 'stroke')
                        .reduce((acc, curr) => {
                            return {
                                ...acc,
                                [curr]: svgShape.getAttribute(curr)
                            }
                        }, {})
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
                if(!this.ignorePolygonSelection) {
                    // if a polygon is already selected, deselect the previous one
                    if(this.selectedPolygon) this.selectPolygon(this.selectedPolygon, false)
                    this.selectedPolygon = event.srcElement
                    this.selectPolygon(this.selectedPolygon, true)
                    event.stopPropagation()
                }
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
                const attributes = {
                    elementType: type,
                    fill: '#000'
                }

                if(type == 'wall') attributes.fill = '#000000'
                if(type == 'wall-sphere') attributes.fill = '#000000'
                if(type == 'gate') {
                    attributes.fill = '#0000FF'
                    attributes.opacity = '0.3'
                }
                if(type == 'windstream') {
                    attributes.fill = '#A020F0'
                    attributes.opacity = '0.3'
                    attributes.xForce = 0.00006
                    attributes.yForce = 0
                }
                if(type == 'ball') attributes.fill = '#FF0000'

                let drawShapeHandler = null
                if(['wall-sphere','ball'].indexOf(type) != -1) drawShapeHandler = new DrawCircleHandler(drawHandler, attributes)
                else drawShapeHandler = new DrawPolygonHandler(drawHandler, attributes)
                drawHandler.setShapeHandler(drawShapeHandler)
            }
    }
}
</script>

<style>

</style>