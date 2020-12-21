<template>
  <div id="app">
    <!-- <input v-model="habitifyKey"/> -->
    <template v-for="habit in habits">
      <li :key="habit.id">
        <div class="name">
          <strong>
            {{ habit.name }}
          </strong>
        </div>
        <div class="goal" v-if="habit.goal">
          <span class="today" v-if="habit.history" style="margin-right:4px">
            {{ getTodaysNumbers(habit.history) }} /
          </span>
          <span class="today-goal">
            {{ habit.goal.value }} {{ habit.goal.unit_type }} {{ habit.goal.periodicity }}
          </span>
          <radial-progress-bar
            style="margin-left: 8px;"
            :diameter="20"
            :completed-steps="getTodaysNumbers(habit.history)"
            :total-steps="habit.goal.value"
            start-color="#2faadc"
            stop-color="#2faadc"
            :stroke-width="3"
            :inner-stroke-width="3"
            inner-stroke-color="#e0dfde"
          />
        </div>
      </li>
    </template>
  </div>
</template>

<script>
import axios from 'axios'
import map from 'lodash/map'
import find from 'lodash/find'
import filter from 'lodash/filter'
import dayjs  from 'dayjs'
import RadialProgressBar from 'vue-radial-progress'

export default {
  name: 'App',
  data() {
    return {
      habitifyKey: '',
      habits: [],
      history: [],
      today: {},
      todayString: ''
    }
  },
  components: {
    RadialProgressBar
  },
  watch: {
    async habitifyKey(newVal) {
      const app = this
      const data = await axios.get('/habits', {
        headers: { Authorization: `${newVal}` },
        params: {
          target_date: app.today.toISOString()
        }
      }).then((res) => {
        return filter(res.data, (habit) => {
          return !habit.is_archived
        })
      })

      const habits = await Promise.all(map(data, async (habit) => {
        const history = await app.getHabitHistory(habit.id)
        console.log(habit, history)

        return Object.assign({
          ...habit,
          history
        })
      }))

      app.habits = habits
    }
  },
  methods: {
    getTargetDateWithoutTime(string) {
      return string ? string.split("T")[0] : null
    },
    getTodaysNumbers(history) {
      const app = this
      const today = app.todayString
      const data = filter(history, (h) => {
        const target = app.getTargetDateWithoutTime(h.targe_date)
        return target == today
      })

      if (data.length == 0) {
        return 0
      }

      console.log('unit type', data, data[0].unit_type, data[0].value)
      switch (data[0].unit_type) {
        case 'J':
          // Convert to KCal
          return Math.round(data[0].value / 4184)
        case 'm':
          // Convert to miles
          return Math.round(data[0].value / 1609)
        default:
          return "blank"
      }
    },
    async getHabitHistory(id) {
      const app = this
      const oneWeekAgo = app.today.subtract(7, 'day').toISOString()
      const today = app.today.toISOString()

      let habit = find(app.habits, { 'id' : id })
      console.log(app.habits, habit)

      const { data } = await axios.get(`/habits/${id}/logs`, {
        headers: { Authorization: `${app.habitifyKey}` },
        params: {
          from: oneWeekAgo,
          to: today,
          limit: 1000
        }
      })

      return data
    }
  },
  mounted() {
    const queryString = window.location.search
    const urlParams = new URLSearchParams(queryString)
    const API_KEY = urlParams.get('api_key')
    this.habitifyKey = API_KEY

    const today = dayjs()
    this.today = today
    this.todayString = today.toISOString().split("T")[0]
  }
}
</script>

<style>
#app {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Helvetica, "Apple Color Emoji", Arial, sans-serif, "Segoe UI Emoji", "Segoe UI Symbol";
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: rgb(55, 53, 47);
  /* margin-top: 60px; */
  background-color: #ffffff;
}

li {
  list-style-type: none;
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.5rem;
}

.goal {
  color: rgba(55, 53, 47, 0.6);
  margin-left: 1rem;
  display: flex;
  flex-direction: row;
  align-items: center;
}
</style>
