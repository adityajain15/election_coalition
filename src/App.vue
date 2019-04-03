<template>
  <v-app>
    <div id="app" class="w-100 pa2 center">
      <div id="graphic" class="w-100">
        <ElectionCoalition :party1="[party1, filter(party1_losses)]" :party2="[party2, filter(party2_losses)]" v-on:tooltip-change="tooltip = $event"/>
        <div id="tooltip" v-if="tooltip.length" :style="tooltipObject" class="hind f5 lh-solid pa2">
          <span class="bold text-red ttc f4">{{tooltip[1][1]}}, {{tooltip[1][0]}}</span>
          <span class="db"><span class="bold">{{tooltip[1][2][0].party}}</span> won this seat with {{tooltip[1][2][0].vote_pct.toFixed(2)}}% of the popular vote</span>
          <span class="db"><span class="bold">{{tooltip[1][2][1].party}}</span> came second with {{tooltip[1][2][1].vote_pct.toFixed(2)}}%</span>
          <span class="db text-red mt2">Click to explore coalitions that would flip this seat</span>
        </div>
        <v-select
          class="w-50"
          v-model="filtering_states"
          :items="states"
          :menu-props="{ maxHeight: '400' }"
          label="Filter by one or more states"
          multiple
          hint="Pick your favorite states"
          persistent-hint
        ></v-select>
      </div>
    </div>
  </v-app>
</template>

<script>
import ElectionCoalition from './components/ElectionCoalition'
import { csv } from 'd3-fetch'
import { nest } from 'd3-collection'
import { filter } from 'minimatch';

export default {
  name: 'app',
  components: {
    ElectionCoalition
  },
  data() {
    return {
      tooltip: [],
      filtering_states: [],
      party1_losses: [],
      party2_losses: [],
      party1: 'BJP',
      party2: 'INC',
      data: {}
    }
  },
  async mounted () {
    const candidates = await csv('./result.csv', (d) => {
      return {
        state: d.state.trim().toLowerCase(),
        name: d.name.trim().toLowerCase(),
        age: +d.age,
        candidate: d.candidate.trim(),
        electors: +d.electors,
        general: +d.general,
        postal: +d.postal,
        sex: d.sex.trim(),
        symbol: d.symbol.trim(),
        total: +d.total,
        party: d.party.trim(),
        category: d.category.trim(),
        vote_pct: +d.vote_pct,
        electors_pct: +d.electors_pct
      }
    })

    this.data = nest()
      .key(d => d.state)
      .key(d => d.name)
      .sortValues((a, b) => parseInt(a.total) > parseInt(b.total) ? -1 : 1)
      .object(candidates)

    this.populate()
  },
  computed: {
    states() {
      const states = []
      for (let i = 0; i < this.party1_losses.length; i++) {
        if(!states.includes(this.party1_losses[i][0])) states.push(this.party1_losses[i][0])
      }
      for (let i = 0; i < this.party2_losses.length; i++) {
        if(!states.includes(this.party2_losses[i][0])) states.push(this.party2_losses[i][0])
      }
      return states
    },
    tooltipObject() {
      const [event, [state, constituency, ballot]] = this.tooltip
      return {
        'top': `${event.y + 5}px`,
        'left': `${event.x + 5}px`
      }
    }
  },
  methods: {
    populate() {
      this.party1_losses = []
      this.party2_losses = []

      for (let state in this.data) {
        for (let constituency in this.data[state]) {
          const seat_data = this.data[state][constituency]
          const [winner, loser, ...others] = seat_data
          if(winner.party === this.party1 && loser.party === this.party2) {
            this.party2_losses.push([state, constituency, this.data[state][constituency]])
          } else if (winner.party === this.party2 && loser.party === this.party1) {
            this.party1_losses.push([state, constituency, this.data[state][constituency]])
          }
        }
      }
    },
    filter(losses) {
      if (!this.filtering_states.length) return losses
      const filtered_losses = []
      for (let i = 0; i < losses.length; i++) {
        if (this.filtering_states.includes(losses[i][0])) filtered_losses.push(losses[i])
      }
      return filtered_losses
    }
  }
}
</script>

<style>
#graphic {
  border: 2px solid black;
  height: 800px;
  position: relative;
}

#tooltip{
  border: 1px solid black;
  position: fixed;
  background: white;
}

.hind {
  font-family: 'Hind', sans-serif;
  font-weight: 400;
}

.bold{
  font-weight: 600;
}

.text-red{
  color: red;
}
</style>
