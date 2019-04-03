<template>
  <svg>
    <defs>
      <g :transform="`translate(${this.width / 2} 0)`">
        <template v-for="(tick, index) in ticks">
          <path :key="`party-1-helperpath-${index}`" :id="`party-1-helperpath-${index}`" :d="getArc(tick, party1_scale)"/>
        </template>
        <template v-for="(tick, index) in ticks_party2">
          <path :key="`party-2-helperpath-${index}`" :id="`party-2-helperpath-${index}`" :d="getArc_party2(tick, party2_scale)"/>
        </template>
      </g>
      <marker id="arrowhead" viewBox="-10 -10 20 20" refX="0" refY="0" markerWidth="20" markerHeight="20" stroke-width="1" orient="auto"><polyline stroke-linejoin="bevel" points="-6.75,-6.75 0,0 -6.75,6.75"></polyline></marker>
    </defs>
    <path :d="`M ${(this.width / 2) - (this.shorter_side)}, ${0} A ${this.shorter_side}, ${this.shorter_side}, 0, 1, 0, ${(this.width / 2) + (this.shorter_side)}, ${0}`"/>
    <g id="party1_circles">
      <template v-for="(loss, index) in party1[1]">
        <circle class="party1" 
          @mouseout="tooltipOff()" 
          @mouseover="tooltip($event, party1[1][index])" 
          @click="hud(party1[0], party1[1][index], true)"
          :key="`party1-circle-${index}`" 
          :cx="width / 2 + (shorter_side * Math.cos(party1_scale(party1_margins[index])))" 
          :cy="0 - (shorter_side * Math.sin(party1_scale(party1_margins[index])))" 
          :lossmargin="party2_margins[index]" 
          r="6"
        />
    </template>
    </g>
    <g id="party2_circles">
      <template v-for="(loss, index) in party2[1]">
        <circle class="party2" 
          @mouseout="tooltipOff()"
          @mouseover="tooltip($event, party2[1][index])"
          @click="hud(party2[0], party2[1][index], false)"
          :key="`party2-circle-${index}`"
          :cx="width / 2 + (shorter_side * Math.cos(party2_scale(party2_margins[index])))" 
          :cy="0 - (shorter_side * Math.sin(party2_scale(party2_margins[index])))" 
          :lossmargin="party2_margins[index]" 
          r="6"
        />
      </template>
    </g>
    <g :transform="`translate(${this.width / 2} ${0})`">
      <template v-for="(tick, index) in ticks">
        <text :key="`party-1-ticktext-${index}`">
          <textPath startOffset="50%" text-anchor="middle" :href="`#party-1-helperpath-${index}`">
            {{ticks[index]}}
          </textPath>
        </text>
      </template>
      <template v-for="(tick, index) in ticks_party2">
        <text :key="`party-2-ticktext-${index}`">
          <textPath startOffset="50%" text-anchor="middle" :href="`#party-2-helperpath-${index}`">
            {{ticks_party2[index]}}
          </textPath>
        </text>
      </template>
    </g>
    <g>
      <template v-for="(arrow,index) in coalitionArrows">
        <path :key="`arrow-${index}`" :d="arrow" marker-end="url(#arrowhead)"/>
      </template>
    </g>
  </svg>
</template>

<script>
import { scaleLinear, scaleSqrt } from 'd3-scale'
import { arc } from 'd3-shape'
import { TweenLite } from 'gsap'

export default {
  name: 'ElectionCoalition',
  props: {
    party1: {
      default: [],
      type: Array
    },
    party2: {
      default: [],
      type: Array
    }
  },
  data() {
    return {
      height: 0,
      width: 0,
      party1_margins: [],
      party2_margins: [],
      largest_loss_interpolated: 100,
      coalitions: [],
      coalitionArrows: []
    }
  },
  mounted () {
    this.width = this.$el.clientWidth
    this.height = this.$el.clientHeight
  },
  computed: {
    shorter_side() {
      return (this.width / 2) < this.height ? (this.width / 2): this.height
    },
    party1_scale() {
      return scaleLinear().domain([this.largest_loss_interpolated, 0]).range([Math.PI, Math.PI * 1.5])
    },
    party2_scale() {
      return scaleLinear().domain([0, this.largest_loss_interpolated]).range([Math.PI * 1.5, Math.PI * 2])
    },
    largest_loss() {
      return this.party1_margins.length && this.party2_margins.length ? Math.max(...this.party1_margins, ...this.party2_margins) : 100
    },
    ticks() {
      return this.party1_scale.ticks()
    },
    ticks_party2() {
      return this.party2_scale.ticks()
    }
  },
  watch: {
    party1(newVal, oldVal) {
      const [new_party, new_losses] = newVal
      const [old_party, old_losses] = oldVal
      this.party1_margins = new_losses.map(d => d[2][0].vote_pct - d[2][1].vote_pct)
    },
    party2(newVal, oldVal) {
      const [party, losses] = newVal
      this.party2_margins = losses.map(d => d[2][0].vote_pct - d[2][1].vote_pct)
    },
    largest_loss(newVal, oldVal) {
      const transitioningObject = { value: oldVal }
      const tween = TweenLite.to(transitioningObject, 2, { 
        value: newVal, 
        onUpdate: this.interpolate,
        onUpdateParams: [transitioningObject]
      })
    }
  },
  methods: {
    interpolate(currentVal) {
      this.largest_loss_interpolated = currentVal.value
    },
    getArc(tick, scale) {
      const textPadding = 20
      //console.log(tick, index)
      return arc()({
        innerRadius: this.shorter_side + textPadding,
        outerRadius: this.shorter_side + textPadding + 1,
        startAngle: (Math.PI/2) - scale(tick +1),
        endAngle: (Math.PI/2) - scale(tick - 1)
      }).split('L')[0]
    },
    getArc_party2(tick, scale) {
      const textPadding = 20
      return arc()({
        innerRadius: this.shorter_side + textPadding,
        outerRadius: this.shorter_side + textPadding + 1,
        startAngle: (Math.PI / 2) - scale(tick - 1),
        endAngle: (Math.PI / 2) - scale(tick + 1),
      }).split('L')[0]
    },
    getX(scale, value, radius) {
      return (this.width / 2) + (radius * Math.cos(scale(value)))
    },
    getY(scale, value, radius) {
      return 0 - (radius * Math.sin(scale(value)))
    },
    hud(party, [state, constituency, ballot], isParty1) {
      const [winner, loser, ...rest] = ballot
      this.coalitions = []
      this.coalitionArrows = []
      //debugger
      //console.log(winner)
      this.getCoalitions([loser], rest, winner.vote_pct)
      console.log(`---${constituency}----`)
      console.log(`Winner has ${winner.vote_pct}%`)
      console.log(this.coalitions.map(d=>d.map(r=>[r.party,r.vote_pct])))
      console.log(`Number of entities on the ballot - ${ballot.length}`)
      console.log(ballot)
      const arrow_radii = this.coalitions.map((d,i)=>(this.shorter_side / (this.coalitions.length+1)) * (i+1))
      console.log(arrow_radii)
      for (let i = 0; i < this.coalitions.length; i++) {
        const revised_voteshare = this.coalitions[i].map(d=>d.vote_pct).reduce((a,b)=>a+b)
        const revised_margin = revised_voteshare - winner.vote_pct
        const winning_margin = winner.vote_pct - loser.vote_pct
        const radii = arrow_radii[i]
        console.log(`${revised_voteshare}`)
        console.log(`${revised_margin}`)
        
        const swing_arc = `M 
          ${this.getX(this.party1_scale, winning_margin, radii)}, ${this.getY(this.party1_scale, winning_margin, radii)} 
          A ${radii}, ${radii}, 
          0, 0, 0, 
          ${this.getX(this.party2_scale, revised_margin, radii)}, ${this.getY(this.party2_scale, revised_margin, radii)}`
        
        this.coalitionArrows.push(swing_arc)
      }
    },
    tooltip(event, data) {
      this.$emit('tooltip-change', [event, data])
    },
    tooltipOff() {
      this.$emit('tooltip-change', [])
    },
    getCoalitions(coalition, all_parties, winning_mark) {
      //console.log(arguments)
      if(this.coalitions.length >= 5) return
      if(!all_parties.length) {
        if (coalition.map(d=>d.vote_pct).reduce((a,b)=>a+b) > winning_mark) {
          this.coalitions.push(coalition)
        }
        return
      } else {
        for(let i = 0; i < all_parties.length; i++) {
          if(this.coalitions.length >= 5) return
          const newCoalition = [...coalition, all_parties[i]]
          if(newCoalition.map(d=>d.vote_pct).reduce((a, b)=>a+b) > winning_mark) {
            //console.log('yes')
            //console.log(newCoalition)
            this.coalitions.push(newCoalition)
          } else {
            this.getCoalitions(newCoalition, all_parties.slice(i + 1), winning_mark)
          }
        }
      }
    }
  }
}
</script>

<style scoped>
svg {
  width: 100%;
  height: 100%;
}
path {
  stroke: black;
  stroke-width: 0.5px;
  fill: transparent;
}

circle {
  cursor: pointer;
  stroke: black;
  stroke-width: 0.1px;
}

circle:hover{
  stroke:#ffcc00;
  stroke-width: 3px;
}

.party1{
  fill: rgb(126, 155, 218);
}

.party2 {
  fill: sandybrown;
}
text {
  font-size: 0.7em;
}
</style>
