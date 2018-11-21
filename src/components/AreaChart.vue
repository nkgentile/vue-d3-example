<template>
  <div class='areachart'>
    <svg :width='width' :height='height'>
      <transition-group tag='g' :css='false' @enter='enter' @leave='leave'>
        <path v-for='d in arcs' class='arc' :key='d.id' :d='d.path' :fill='d.fill' stroke='#fff'
          @mouseenter='() => hovered = d' @mouseleave='() => hovered = null' />
      </transition-group>
      <g ref='xAxis' :transform='`translate(0, ${y0})`' />
      <g ref='yAxis' :transform='`translate(${margin.left}, 0)`' />
    </svg>
    <div v-if='hovered' class='hovered' :style='{
      top: `${hovered.y + 20}px`,
      left: `${hovered.x}px`,
      }'>
      <strong>title</strong> {{ hovered.data.title }}<br />
      <strong>date</strong> {{ formatDate(hovered.data.date) }}<br />
      <strong>metascore</strong> {{ hovered.data.score }}<br />
      <strong>box office</strong> {{ formatBox(hovered.data.boxOffice) }}<br />
    </div>
  </div>
</template>

<script>
import {
  chain,
} from 'lodash';
import {
  area,
  axisBottom,
  axisLeft,
  curveCatmullRom,
  extent,
  format,
  interpolateViridis,
  median,
  scaleLinear,
  scaleSequential,
  scaleTime,
  select,
  timeFormat,
  timeMonth,
} from 'd3';
import {TweenLite} from 'gsap';

const width = 1000;
const height = 300;
const margin = {top: 0, right: 0, bottom: 20, left: 60};

export default {
  name: 'areachart',
  props: ['movies', 'filtered'],
  data() {
    return {
      width, height, margin,
      arcs: [],
      medianBox: 0,
      y0: 0,
      hovered: null,
    }
  },
  created() {
    this.xScale = scaleTime().range([margin.left, width - margin.right]);
    this.yScale = scaleLinear().range([height - margin.bottom, margin.top]);
    this.colorScale = scaleSequential(interpolateViridis);

    this.areaGen = area().curve(curveCatmullRom);

    this.xAxis = axisBottom().scale(this.xScale).tickFormat(this.format);
    this.yAxis = axisLeft().scale(this.yScale)
      .tickSizeOuter(0)
      .tickFormat(d => `${format('$')(parseInt(d / 1000000))}M`);
  },
  watch: {
    movies: function() {
      this.calculateScales()
      select(this.$refs.xAxis).call(this.xAxis)
      select(this.$refs.yAxis).call(this.yAxis)
        .select('.domain').remove();
    },
    filtered: function() {
      this.calculateData();
    }
  },
  methods: {
    calculateScales: function() {
      if (!this.movies.length) return;

      // calculate median box office
      this.medianBox = median(this.movies, d => d.boxOffice);

      // colors
      const colorDomain = extent(this.movies, d => d.score);
      this.colorScale.domain(colorDomain).nice();

      // x scale with dates
      const [minDate, maxDate] = extent(this.movies, d => d.date);
      this.xScale.domain([
        timeMonth.offset(minDate, -2),
        timeMonth.offset(maxDate, 2),
      ]);

      // y scale with boxOffice - medianBox
      const yExtent = extent(this.movies, d => d.boxOffice - this.medianBox);
      this.yScale.domain(yExtent);
      this.y0 = this.yScale(0);

      // update area generater
      this.areaGen.y0(this.y0);
    },
    calculateData: function() {
      if (!this.filtered.length) return;

      // calculate arcs from the filtered movies
      this.arcs = chain(this.filtered)
        // put biggest arcs in the background
        .sortBy(d => -Math.abs(d.boxOffice - this.medianBox))
        .map(d => {
          const x = this.xScale(d.date);
          const y = this.yScale(d.boxOffice - this.medianBox);
          // for each arc, just need d & fill
          return {
            id: d.id,
            path: this.areaGen([
              [this.xScale(timeMonth.offset(d.date, -2)), this.y0],
              [x, y],
              [this.xScale(timeMonth.offset(d.date, 2)), this.y0],
            ]),
            fill: this.colorScale(d.score),
            data: d, x, y,
          };
        }).value();
    },
    enter: function (el, done) {
      TweenLite.fromTo(el, 0.25,
        {scaleY: 0, svgOrigin: `0 ${this.y0}`},
        {scaleY: 1, onComplete: done}
      );
    },
    leave: function (el, done) {
      TweenLite.fromTo(el, 0.25,
        {scaleY: 1},
        {scaleY: 0, svgOrigin: `0 ${this.y0}`, onComplete: done}
      );
    },
    formatDate: function(date) {
      return timeFormat('%b %d, %Y')(date);
    },
    formatBox: function(box) {
      return format("$,.0f")(box);
    }
  }
}
</script>

<style>
.areachart {
  position: relative;
}

.arc {
  cursor: pointer;
}

.hovered {
  position: absolute;
  padding: 5px 10px;
  background-color: #fff;
  box-shadow: 0 0 5px #cfcfcf;
  transform: translate(-50%);
  pointer-events: none;
  line-height: 1.6;
  width: 240px;
}
</style>
