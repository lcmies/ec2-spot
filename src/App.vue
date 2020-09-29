<template>
  <div id="app">
    <VueGoodTable
      :columns="fields"
      :rows="rows"
      :pagination-options="{
        enabled: true,
        perPage: 20
      }"
    ></VueGoodTable>
  </div>
</template>

<script>
import { VueGoodTable } from 'vue-good-table'
import axios from 'axios'

import 'vue-good-table/dist/vue-good-table.css'

export default {
  name: 'App',
  components: {
    VueGoodTable
  },
  data() {
    return {
      interruptRanges: [],
      rawData: [],
      ready: false
    }
  },
  computed: {
    fields() {
      return [
        {
          label: "region",
          field: "region",
          filterOptions: {
            enabled: true,
            filterDropdownItems: this.regionSelect
          }
        },
        {
          label: "os",
          field: "os",
          filterOptions: {
            enabled: true,
            filterDropdownItems: ['Linux', 'Windows']
          }
        },
        {
          label: "type",
          field: "type",
          filterOptions: {
            enabled: true
          }
        },
        {
          label: "cpu",
          field: "cpu",
          type: "number",
          filterOptions: {
            enabled: true,
            filterDropdownItems: [1, 2, 4, 8, 16, 32, 64],
            filterFn: (data, filterString) => data >= parseInt(filterString)
          }
        },
        {
          label: "mem",
          field: "mem",
          type: "number",
          filterOptions: {
            enabled: true,
            filterDropdownItems: [0.5, 1, 2, 4, 8, 16, 32, 64, 128],
            filterFn: (data, filterString) => data >= parseInt(filterString)
          }
        },
        {
          label: "price",
          field: "price",
          type: "number"
        },
        {
          label: "interrupt",
          field: "interrupt",
          type: "number",
          formatFn: (val) => this.interruptRanges[val].label
        }
      ]
    },
    rows() {
      return this.$data.rawData
    },
    regionSelect() {
      return Object.keys(this.$data.rawData.reduce((obj, d) => {
        obj[d.region] = 0
        return obj
      }, {})).sort()
    }
  },
  mounted() {
    window.AWS = {}
    const dictSrc = document.createElement('script')
    dictSrc.src = 'https://a0.awsstatic.com/chrome/js/1.0.537/pricing/dictionary-en_US.js'
    document.querySelector('head').appendChild(dictSrc)

    axios
      .get('https://spot-bid-advisor.s3.amazonaws.com/spot-advisor-data.json')
      .then((res) => {
        this.parseSpotAdvisor(res.data)

        window.callback = (data) => {
          this.parseSpotPrice(data)
          this.$data.ready = true
        }

        const spotSrc = document.createElement('script')
        spotSrc.src = 'https://website.spot.ec2.aws.a2z.com/spot.js'
        document.querySelector('head').appendChild(spotSrc)
      })
  },
  methods: {
    parseSpotAdvisor(data) {
      this.$data.interruptRanges = data.ranges

      let instanceTypes = data.instance_types
      for(const region in data.spot_advisor) {
        for(const instanceType in data.spot_advisor[region]['Windows']) {
          this.$data.rawData.push({
            region:    window.AWS.Dict[region] || region,
            os:        "Windows",
            type:      instanceType,
            cpu:       parseInt(instanceTypes[instanceType].cores),
            mem:       parseFloat(instanceTypes[instanceType].ram_gb),
            price:     undefined,
            interrupt: data.spot_advisor[region]['Windows'][instanceType].r
          })
        }

        for(const instanceType in data.spot_advisor[region]['Linux']) {
          this.$data.rawData.push({
            region:    window.AWS.Dict[region] || region,
            os:        "Linux",
            type:      instanceType,
            cpu:       parseInt(instanceTypes[instanceType].cores),
            mem:       parseFloat(instanceTypes[instanceType].ram_gb),
            price:     undefined,
            interrupt: data.spot_advisor[region]['Linux'][instanceType].r
          })
        }
      }

    },
    parseSpotPrice(data) {
      const info = data.config.regions.reduce((obj, regionInfo) => {
        obj[window.AWS.Dict[regionInfo.region] || regionInfo.region] = regionInfo.instanceTypes.reduce((arr, type) => arr.concat(type.sizes), []).reduce((_obj, sizeInfo) => {
          _obj[sizeInfo.size] = sizeInfo.valueColumns.reduce((__obj, valueInfo) => {
            __obj[valueInfo.name == 'linux' ? "Linux" : "Windows"] = parseFloat(valueInfo.prices.USD)
            return __obj
          }, {})
          return _obj
        }, {})
        return obj
      }, {})

      this.$data.rawData.forEach((d, idx) => {
        if (!info[d.region]) return
        this.$data.rawData[idx].price = info[d.region][d.type][d.os]
      })
    }
  }
}
</script>

<style lang="scss">
html, body {
  margin: 0;
  padding: 0;
}

#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}
</style>
