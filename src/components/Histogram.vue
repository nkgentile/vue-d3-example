<template>
  <svg class="histogram" :width="width" :height="height">
    <g class="bars">
      <rect
        v-for="({
          x, y, width, height, fill,
        }, i) in bars"
        :key="i"
        v-bind="{
          x,
          y,
          width,
          height,
          fill,
          stroke
        }"
      ></rect>
    </g>
    <g class="axis" :transform="`translate(0, ${height - margin.bottom})`">
      <g class="domain">
        <path :d="axis" fill="none" stroke="black"></path>
        <line
          v-for="(tick, i) in ticks"
          :key="i"
          class="tick"
          :x1="scaleX(tick)"
          :x2="scaleX(tick)"
          :y2="10"
          stroke="black"
        ></line>
      </g>
      <g class="ticks" transform="translate(0, 20)">
        <template v-for="tick in ticks">
          <text
            :key="tick"
            class="label"
            :x="scaleX(tick)"
            y="0"
            text-anchor="middle"
          >{{ format(tick) }}</text>
        </template>
      </g>
    </g>
    <g ref="brush"></g>
  </svg>
</template>

<script>
  import { extent, histogram, max, median } from "d3-array";
  import { scaleLinear, scaleSequential } from "d3-scale";
  import { brushX, brushSelection } from "d3-brush";
  import { select } from "d3-selection";
  import { interpolateViridis } from "d3-scale-chromatic";
  import { TweenMax } from "gsap";

  const width = 500;
  const height = 120;
  const margin = { top: 20, right: 20, bottom: 20, left: 20 };

  export default {
    name: "histogram",
    props: {
      movies: {
        type: Array,
        default: () => []
      },
      filtered: {
        type: Array,
        default: () => []
      },
      id: {
        type: String
      },
      format: {
        type: Function,
        default: a => a
      },
      updateFilters: {
        type: Function,
        default: () => {}
      }
    },
    data() {
      return {
        width,
        height,
        margin
      };
    },
    computed: {
      domainX() {
        return extent(this.movies, d => d[this.id]);
      },
      scaleX() {
        return scaleLinear()
          .range([this.margin.left, this.width - this.margin.right])
          .domain(this.domainX)
          .nice();
      },
      ticks() {
        return this.scaleX.ticks();
      },
      axis() {
        const range = this.scaleX.range();
        return `M${range[0] + 0.5},6V0.5H${range[range.length - 1] + 0.5}V6`;
      },
      maxY() {
        const bins = this.histogram(this.movies);
        return max(bins, d => d.length);
      },
      scaleY() {
        return scaleLinear()
          .range([this.height - this.margin.bottom, this.margin.top])
          .domain([0, this.maxY]);
      },
      domainZ() {
        return extent(this.movies, d => d.score);
      },
      scaleZ() {
        return scaleSequential(interpolateViridis)
          .domain(this.domainZ)
          .nice();
      },
      histogram() {
        return histogram()
          .domain(this.domainX)
          .thresholds(this.scaleX.ticks(50))
          .value(d => d[this.id]);
      },
      bars() {
        if (!this.filtered.length) return;

        // calculate rect bar for each bin
        return this.histogram(this.filtered).map((d, i) => {
          const { x0, x1, length } = d;
          const x = this.scaleX(x0);
          const y = this.scaleY(length);
          const fill = this.scaleZ(median(d, d => d.score) || 0);

          return {
            id: i,
            x,
            y,
            width: this.scaleX(x1) - x,
            height: height - margin.bottom - y,
            fill
          };
        });
      }
    },
    watch: {
      bars: {
        handler(newBars, oldBars) {
          // eslint-disable-next-line
          console.log(newBars);
          return new TweenMax(
            this.bars,
            25,
            {
              cycle: {
                height: i => (oldBars ? oldBars[i].height : 0)
              }
            },
            {
              cycle: {
                height: i => newBars[i].height
              }
            }
          );
        }
      }
    },
    created() {
      this.brush = brushX()
        .extent([
          [margin.left, margin.top],
          [width - margin.right, height - margin.bottom]
        ])
        .on("brush", this.brushEnd)
        .on("end", this.brushEnd);
    },
    mounted() {
      select(this.$refs.brush).call(this.brush);
    },
    methods: {
      brushEnd: function() {
        let bounds = null;
        if (event.selection) {
          const [x1, x2] = event.selection;
          bounds = [this.scaleX.invert(x1), this.scaleX.invert(x2)];
        }
        this.updateFilters({ [this.id]: bounds });
      }
    }
  };
</script>

<style scoped>
</style>