<script lang="ts">
import * as d3 from "d3";
import Data from '../../data/demo.json'; /* Example of reading in data directly from file */
import axios from 'axios';
import { isEmpty, debounce } from 'lodash';
import { state } from '../state';
import { Bar, ComponentSize, Margin, Messages } from '../types';
import { useEventEmitter } from '../emitter';

const emitter = useEventEmitter();

// A "extends" B means A inherits the properties and methods from B.
interface SentimentCategory extends Bar{
    category: string;
    value: number;
}

// Computed property: https://vuejs.org/guide/essentials/computed.html
// Lifecycle in vue.js: https://vuejs.org/guide/essentials/lifecycle.html#lifecycle-diagram

export default {
    
    data() {
        // Here we define the local states of this component. If you think the component as a class, then these are like its private variables.
        return {
            bars: [] as SentimentCategory[], // "as <Type>" is a TypeScript expression to indicate what data structures this variable is supposed to store.
            size: { width: 0, height: 0 } as ComponentSize,
            margin: {left: 40, right: 20, top: 20, bottom: 60} as Margin,
        }
    },
    computed: {
        // Re-render the chart whenever the window is resized or the data changes (and data is non-empty)
        rerender() {
            return (!isEmpty(this.bars)) && this.size
        }
    },
    // Anything in here will only be executed once.
    // Refer to the lifecycle in Vue.js for more details, mentioned at the very top of this file.
    async created() {
        // const rawData = await d3.csv("../../data/proc/palm-2.csv");
        
        this.updateChart();
    },
    methods: {
        async updateChart() {
            const selectedPoint = state.selections;
            const content = selectedPoint.get("content");

            const serverUrl = 'http://localhost:8000/endpoint';

            // Input string for the GET request
            // Make a GET request using Axios
            await axios.get(serverUrl, {
                params: {
                    input_string: content
                }
            })
            .then(response => {
                // Handle the response from the server
                this.bars = response.data as SentimentCategory[];
                this.initChart();
            })
            .catch(error => {
                // Handle any errors that occurred during the request
                console.error('Error:', error);
            });
        
        },
        onResize() {  // record the updated size of the target element
            let target = this.$refs.barContainer as HTMLElement
            if (target === undefined) return;
            this.size = { width: target.clientWidth, height: target.clientHeight };
        },
        initChart() {
            // remove all the elements in the chart
            d3.select('#bar-svg').selectAll('*').remove()
            // select the svg tag so that we can insert(render) elements, i.e., draw the chart, within it.
            let chartContainer = d3.select('#bar-svg')
            // Here we compute the [min, max] from the data values of the attributes that will be used to represent x- and y-axis
            let yExtents = d3.extent(this.bars.map((d: SentimentCategory) => d.value)) as [number, number];

            // let yExtents = d3.extent(this.bars.map((d: SentimentCategory) => d.value as number)) as [number, number]
            // This is to get the unique categories from the data using Set, then store in an array.
            // let xCategories: string[] = [ ...new Set(Object.keys(this.bars).map((d: SentimentCategory) => d.category as string))]
            let xCategories: string[] = [ ...new Set(this.bars.map((d: SentimentCategory) => d.category as string))]

            // We need a way to map our data to where it should be rendered within the svg (in screen pixels), based on the data value, 
            //      so the extents and the unique values above help us define the limits.
            // Scales are just like mapping functions y = f(x), where x refers to domain, y refers to range. 
            //      In our case, x should be the data, y should be the screen pixels.
            // We have the margin here just to leave some space
            // In viewport (our screen), the leftmost side always refer to 0 in the horizontal coordinates in pixels (x). 
            let xScale = d3.scaleBand()
                .rangeRound([this.margin.left, this.size.width - this.margin.right])
                .domain(xCategories)
                .padding(0.1) // spacing between the categories

            // In viewport (our screen), the topmost side always refer to 0 in the vertical coordinates in pixels (y). 
            let yScale = d3.scaleLinear()
                .range([this.size.height - this.margin.bottom, this.margin.top]) //bottom side to the top side on the screen
                .domain([0, yExtents[1]]) // This is based on your data, but if there is a natural value range for your data attribute, you should follow
                // e.g., it is natural to define [0, 100] for the exame score, or [0, <maxVal>] for counts.

            // There are other scales such as scaleOrdinal,
                // whichever is appropriate depends on the data types and the kind of visualizations you're creating.

            // This following part visualizes the axes along with axis labels.
            // Check out https://observablehq.com/@d3/margin-convention?collection=@d3/d3-axis for more details
            const xAxis = chartContainer.append('g')
                .attr('transform', `translate(0, ${this.size.height - this.margin.bottom})`)
                .call(d3.axisBottom(xScale))

            const yAxis = chartContainer.append('g')
                .attr('transform', `translate(${this.margin.left}, 0)`)
                .call(d3.axisLeft(yScale))

            const yLabel = chartContainer.append('g')
                .attr('transform', `translate(${10}, ${this.size.height / 2}) rotate(-90)`)
                .append('text')
                .text('Value')
                .style('font-size', '.8rem')

            const xLabel = chartContainer.append('g')
                .attr('transform', `translate(${this.size.width / 2 - this.margin.left}, ${this.size.height - this.margin.top - 5})`)
                .append('text')
                .text('Categories')
                .style('font-size', '.8rem')
            
            // "g" is grouping element that does nothing but helps avoid DOM looking like a mess
            // We iterate through each <CategoricalBar> element in the array, create a rectangle for each and indicate the coordinates, the rectangle, and the color.
            const bars = chartContainer.append('g')
                .selectAll('rect')
                .data<SentimentCategory>(this.bars) // TypeScript expression. This always expects an array of objects.
                .join('rect')
                // specify the left-top coordinate of the rectangle
                .attr('x', (d: SentimentCategory) => xScale(d.category) as number)
                .attr('y', (d: SentimentCategory) => yScale(d.value) as number)
                // specify the size of the rectangle
                .attr('width', xScale.bandwidth())
                .attr('height', (d: SentimentCategory) => Math.abs(yScale(0) - yScale(d.value))) // this substraction is reversed so the result is non-negative
                .attr('fill', 'teal')

            // For transform, check out https://www.tutorialspoint.com/d3js/d3js_svg_transformation.htm, but essentially we are adjusting the positions of the selected elements.
            const title = chartContainer.append('g')
                .append('text') // adding the text
                .attr('transform', `translate(${this.size.width / 2}, ${this.size.height - this.margin.top + 5})`)
                .attr('dy', '0.5rem') // relative distance from the indicated coordinates.
                .style('text-anchor', 'middle')
                .style('font-weight', 'bold')
                .text('Sentiment Analysis of the message') // text content
        }
    },
    watch: {
        rerender(newSize) {
            if (!isEmpty(newSize)) {
                d3.select('#bar-svg').selectAll('*').remove() // Clean all the elements in the chart
                this.initChart()
            }
        }
    },
    // The following are general setup for resize events.
    mounted() {
        emitter.$on('update-chart', this.updateChart);
        window.addEventListener('resize', debounce(this.onResize, 100)) 
        this.onResize()
    },
    beforeDestroy() {
       emitter.$off('update-chart', this.updateChart);
       window.removeEventListener('resize', this.onResize)
    }
}

</script>

<!-- "ref" registers a reference to the HTML element so that we can access it via the reference in Vue.  -->
<!-- We use flex (d-flex) to arrange the layout-->
<template>
    <div class="chart-container d-flex" ref="barContainer">
        <svg id="bar-svg" width="100%" height="100%">
            <!-- all the visual elements we create in initChart() will be inserted here in DOM-->
        </svg>
    </div>
</template>

<style scoped>
.chart-container{
    height: 100%;
}
</style>
