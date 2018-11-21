<template>
  <svg class='histogram' :width='width' :height='height'>
    <g class='bars'>
      <rect v-for='d in bars' :key='d.id' :x='d.x' :width='d.width'
        :y='d.y' :height='d.height' :fill=d.fill :stroke='d.fill' />
    </g>
    <g ref='xAxis' :transform='`translate(0, ${height - margin.bottom})`' />
    <g ref='brush' />
  </svg>
</template>

<script>
import {
  axisBottom,
  brushX,
  extent,
  histogram,
  interpolateViridis,
  max,
  median,
  scaleLinear,
  scaleSequential,
  select,
} from 'd3';
import {TweenMax} from 'gsap';

const width = 500
const height = 120
const margin = {top: 20, right: 20, bottom: 20, left: 20}

export default {
  name: 'histogram',
  props: ['movies', 'filtered', 'id', 'format', 'updateFilters'],
  data() {
    return {
      width, height, margin,
      bars: [],
    }
  },
  created() {
    this.xScale = scaleLinear().range([margin.left, width - margin.right]);
    this.histogram = histogram();

    this.yScale = scaleLinear().range([height - margin.bottom, margin.top]);
    this.colorScale = scaleSequential(interpolateViridis);

    this.xAxis = axisBottom().scale(this.xScale).tickFormat(this.format);
    this.brush = brushX().extent([
      [margin.left, margin.top],
      [width - margin.right, height - margin.bottom]
    ]).on('brush', this.brushEnd)
    .on('end', this.brushEnd)
  },
  mounted() {
    select(this.$refs.brush).call(this.brush);
  },
  watch: {
    movies: function() {
      this.calculateScales();
      select(this.$refs.xAxis).call(this.xAxis);
    },
    filtered: function() {
      this.calculateData();
    }
  },
  methods: {
    calculateScales: function() {
      if (!this.movies.length) return;

      // colors
      const colorDomain = extent(this.movies, d => d.score);
      this.colorScale.domain(colorDomain).nice();

      // x is the attribute
      const xDomain = extent(this.movies, d => d[this.id]);
      this.xScale.domain(xDomain).nice();

      // get the bins
      this.histogram.domain(this.xScale.domain())
        .thresholds(this.xScale.ticks(50))
        .value(d => d[this.id]);
      const bins = this.histogram(this.movies);

      // calculate y from the bins
      const yMax = max(bins, d => d.length);
      this.yScale.domain([0, yMax]);
    },
    calculateData: function() {
      if (!this.filtered.length) return;

      // use filtered for bins now that scales are calculated
      const bins = this.histogram(this.filtered);

      // calculate rect bar for each bin
      this.bars = bins.map((d, i) => {
        const {x0, x1} = d;
        const x = this.xScale(x0);
        const y = this.bars[i] ? this.bars[i].toY : this.yScale(0);
        const toY = this.yScale(d.length);
        const fill = this.colorScale(median(d, d => d.score) || 0);

        return {
          id: i,
          x,
          width: this.xScale(x1) - x,
          y, toY,
          height: height - margin.bottom - y,
          toHeight: height - margin.bottom - toY,
          fill,
        };
      });

      // and then animate them
      TweenMax.staggerFromTo(this.bars, 0.25, {
        // from
        cycle: {
          y: (i) => this.bars[i].y,
          height: (i) => this.bars[i].height,
        }
      }, {
        // to
        cycle: {
          y: (i) => this.bars[i].toY,
          height: (i) => this.bars[i].toHeight,
        },
      })
    },
    brushEnd: function() {
      let bounds = null;
      if (event.selection) {
        const [x1, x2] = event.selection;
        bounds = [
          this.xScale.invert(x1),
          this.xScale.invert(x2),
        ]
      }
      this.updateFilters({[this.id]: bounds});
    },
  }
}
</script>

<style scoped>
</style>
