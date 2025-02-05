<template>
  <dashboard-card
    :loading="loading"
    type="heatmap"
    :options="chartOptions"
    :series="series"
    title="Heatmap"
  />
</template>

<script>
import { countBy, isArray, mergeWith, forEach, map, find, sortBy } from "lodash"
import { parseISO } from "date-fns"
import locale from "date-fns/esm/locale/en-US"

import DashboardCard from "@/dashboard/DashboardCard.vue"
import DashboardUtils from "@/dashboard/utils"

export default {
  name: "IncidentHeatMapChartCard",

  props: {
    value: {
      type: Object,
      default: function() {
        return {}
      }
    },
    loading: {
      type: [String, Boolean],
      default: function() {
        return false
      }
    }
  },
  components: {
    DashboardCard
  },

  computed: {
    chartOptions() {
      return {
        chart: {
          height: 350,
          type: "heatmap",
          toolbar: {
            show: false
          }
        },
        colors: DashboardUtils.defaultColorTheme(),
        responsive: [
          {
            options: {
              legend: {
                position: "top"
              }
            }
          }
        ],
        dataLabels: {
          enabled: false
        },

        xaxis: {
          categories: this.categoryData || [],
          title: {
            text: "Month"
          }
        }
      }
    },
    series() {
      let series = []
      let weekdays = this.weekdays

      forEach(this.value, function(value) {
        let dayCounts = map(
          countBy(value, function(item) {
            return parseISO(item.reported_at).toLocaleString("default", { weekday: "short" })
          }),
          function(value, key) {
            return { name: key, data: [value] }
          }
        )

        // fill in any gaps
        forEach(weekdays, function(weekday) {
          let found = find(dayCounts, { name: weekday })
          if (!found) {
            dayCounts.push({ name: weekday, data: [0] })
          }
        })

        let sortedDayCounts = sortBy(dayCounts, function(obj) {
          return weekdays.indexOf(obj.name)
        })
        series = mergeWith(series, sortedDayCounts, function(objValue, srcValue) {
          if (isArray(objValue)) {
            return objValue.concat(srcValue)
          }
        })
      })
      // sort
      series = sortBy(series, function(obj) {
        return weekdays.indexOf(obj.name)
      })
      return series
    },
    categoryData() {
      return Object.keys(this.value)
    },
    weekdays() {
      return [...Array(7).keys()].map(i => locale.localize.day(i, { width: "abbreviated" }))
    }
  }
}
</script>
