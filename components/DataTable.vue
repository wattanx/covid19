<template>
  <data-view :title="title" :title-id="titleId" :date="date" :url="url">
    <template v-slot:button>
      <span />
    </template>
    <v-data-table
      :headers="chartData.headers"
      :items="chartData.datasets"
      :height="240"
      fixed-header
      :mobile-breakpoint="0"
      :custom-sort="customSort"
      :footer-props="{
        'items-per-page-options': [15, 30, 50, 100, 200, 300, -1],
        'items-per-page-text': $t('1ページ当たり')
      }"
      class="cardTable"
    >
      <template slot="footer.page-text" slot-scope="props">
        {{
          $t('{itemsLength} 項目中 {pageStart} - {pageStop}', {
            itemsLength: props.itemsLength,
            pageStart: props.pageStart,
            pageStop: props.pageStop
          })
        }}
      </template>
    </v-data-table>
    <div class="note">
      {{ $t('※退院には死亡退院を含む') }}
    </div>
    <template v-slot:infoPanel>
      <data-view-basic-info-panel
        :l-text="info.lText"
        :s-text-list="[info.sText]"
        :unit="$t('人')"
      />
    </template>
    <template v-slot:footer>
      <open-data-link :url="url" />
    </template>
  </data-view>
</template>

<style lang="scss">
.cardTable {
  &.v-data-table {
    th {
      padding: 8px 10px;
      height: auto;
      border-bottom: 1px solid $gray-4;
      white-space: nowrap;
      color: $gray-2;
      font-size: 12px;
      &.text-center {
        text-align: center;
      }
    }
    tbody {
      tr {
        color: $gray-1;
        th {
          font-weight: normal;
        }
        td {
          padding: 8px 10px;
          height: auto;
          font-size: 12px;
          &.text-center {
            text-align: center;
          }
        }
        &:nth-child(odd) {
          td {
            background: rgba($gray-4, 0.3);
          }
        }
        &:not(:last-child) {
          td:not(.v-data-table__mobile-row) {
            border: none;
          }
        }
      }
    }
    .v-select {
      margin-left: 10px;
    }
    &:focus {
      outline: dotted $gray-3 1px;
    }
  }
  .v-data-table__wrapper {
    box-shadow: 0 -20px 12px -12px #0003 inset;
  }
  &.v-data-footer__pagination {
    margin-left: 0;
    margin-right: 5px;
  }
}
.note {
  padding: 8px;
  font-size: 12px;
  color: #808080;
}
// FIXME: スタイルの影響が全体に及んでいるので修正が必要
.v-menu__content {
  width: 60px;
  .v-list-item {
    padding: 0 8px;
  }
}
</style>

<script>
import DataView from '@/components/DataView.vue'
import DataViewBasicInfoPanel from '@/components/DataViewBasicInfoPanel.vue'
import OpenDataLink from '@/components/OpenDataLink.vue'

const excludeAges = ['就学児', '未就学児']

export default {
  components: { DataView, DataViewBasicInfoPanel, OpenDataLink },
  props: {
    title: {
      type: String,
      default: ''
    },
    titleId: {
      type: String,
      default: ''
    },
    chartData: {
      type: Object,
      default: () => {}
    },
    date: {
      type: String,
      default: ''
    },
    info: {
      type: Object,
      required: false,
      default: () => {}
    },
    url: {
      type: String,
      required: false,
      default: ''
    }
  },
  methods: {
    customSort(items, index, isDescending) {
      if (isDescending[0] === undefined) return items
      if (index[0] === '年代') {
        return this.createSortAgeData(items, index[0], isDescending[0])
      } else if (index[0] === '日付') {
        return this.createSortDateData(items, index[0], isDescending[0])
      } else {
        items.sort((a, b) => {
          if (b[index[0]] < a[index[0]]) {
            return isDescending[0] ? -1 : 1
          } else {
            return isDescending[0] ? 1 : -1
          }
        })
      }
      return items
    },
    createSortDateData(items, index, isDescending) {
      return items.sort((a, b) => {
        const aDate = a[index].split('/').map(d => parseInt(d))
        const bDate = b[index].split('/').map(d => parseInt(d))

        let comparison = aDate[1] > bDate[1] ? 1 : -1
        if (aDate[0] > bDate[0]) {
          comparison = 1
        } else if (aDate[0] < bDate[0]) {
          comparison = -1
        }
        return isDescending ? comparison * -1 : comparison
      })
    },
    createSortAgeData(items, index, isDescending) {
      const excludeItems = {}
      const investigatingItems = []
      // 翻訳後のテキストを取得する
      const translatedAges = excludeAges.map(v => this.$t(v))

      const filterItems = items.filter(item => {
        if (translatedAges.includes(item[index])) {
          excludeItems[item[index]] = excludeItems[item[index]] || []
          excludeItems[item[index]].push(item)
          return false
        } else if (item[index] === this.$t('調査中')) {
          investigatingItems.push(item)
          return false
        } else {
          return true
        }
      })

      filterItems.sort((a, b) => {
        const aInt = parseInt(a[index].substring(0, a[index].length - 1), 10)
        const bInt = parseInt(b[index].substring(0, b[index].length - 1), 10)

        if (isDescending) {
          return aInt - bInt
        } else {
          return bInt - aInt
        }
      })

      translatedAges.forEach(v => {
        if (!excludeItems[v]) {
          return
        }

        excludeItems[v].forEach(item => {
          filterItems[isDescending ? 'unshift' : 'push'](item)
        })
      })

      investigatingItems.forEach(item => {
        filterItems[isDescending ? 'push' : 'unshift'](item)
      })

      return filterItems
    }
  }
}
</script>
