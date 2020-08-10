<template>
  <div class="hello">
    <h1>Corona Virus Daily Increases - {{ currentDayFormatted }}</h1>
    <CoronaMap
      v-if="coronaData"
      :data="currentDay"
    />
    <button @click="timelinePlay">{{ buttonText }}</button>
  </div>
</template>

<script>
import moment from 'moment'
import axios from 'axios'
import stateCodes from '../modules/state-codes'
import CoronaMap from '../components/CoronaMap'

export default {
  name: 'Index',

  data () {
    return {
      coronaData: null,
      currentIndex: 0,
      stateData: null,
      currentDate: null,
      playInterval: null,
      isPlaying: false,
      startDate: '20200301',
      endDate: null
    }
  },

  components: {
    CoronaMap
  },

  computed: {
    currentDay () {
      return this.coronaData[this.currentDate]
    },

    currentDayFormatted () {
      return moment(this.currentDate).format('MMMM Do, YYYY')
    },

    buttonText () {
      return this.isPlaying ? 'Stop' : 'Play'
    }
  },

  methods: {
    async fetchData () {
      const rawStateData = await axios.get('https://datausa.io/api/data?drilldowns=State&measures=Population&year=latest')
        .then(res => {
          return res.data.data
        })

      this.stateData = this.normalizeStateData(rawStateData)

      axios.get('https://covidtracking.com/api/states/daily')
        .then(res => {
          this.normalizeCoronaData(res.data)
        })
    },

    normalizeCoronaData (data) {
      const normalizedData = {}

      data.forEach(statistic => {
        if (typeof normalizedData[statistic.date] === 'undefined') {
          normalizedData[statistic.date] = {}
        }

        if (typeof normalizedData[statistic.date][statistic.state] === 'undefined') {
          normalizedData[statistic.date][statistic.state] = {}
        }

        normalizedData[statistic.date][statistic.state] = statistic
        normalizedData[statistic.date][statistic.state].averageDailyCasePercent = this.getIncreaseRate(statistic)
      })

      const dateArray = Object.keys(normalizedData)

      this.endDate = dateArray[dateArray.length - 1]

      this.coronaData = normalizedData
    },

    normalizeStateData (data) {
      const normalizedData = {}

      data.forEach(state => {
        const stateCode = stateCodes[state.State]

        if (typeof stateCode !== 'undefined') {
          normalizedData[stateCode] = state
        }
      })

      return normalizedData
    },

    getIncreaseRate (statistic) {
      if (typeof this.stateData[statistic.state] === 'undefined') {
        return 0
      }

      const statePopulation = this.stateData[statistic.state].Population

      if (typeof statePopulation === 'undefined') {
        return 0
      }

      const positiveByPopulation = statistic.positiveIncrease / statePopulation

      const normalizedDaily = positiveByPopulation * 100000 / 56 // postive cases per 100,000 in given state

      return normalizedDaily
    },

    timelinePlay () {
      if (!this.isPlaying) {
        this.playInterval = setInterval(() => {
          this.currentDate = moment(this.currentDate).add(1, 'days').format('YYYYMMDD')

          if (this.currentDate === this.endDate) {
            clearInterval(this.playInterval)
            this.isPlaying = false
            this.currentDate = this.startDate
          }
        }, 500)
      } else {
        clearInterval(this.playInterval)
      }

      this.isPlaying = !this.isPlaying
    }
  },

  mounted () {
    this.currentDate = this.startDate
    this.fetchData()
  }
}
</script>
