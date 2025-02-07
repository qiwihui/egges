<template>
  <div>
    <v-card-text>
      <v-row>
        <v-col>
          <new-index @reloadIndices="emitReloadIndices"/>
        </v-col>
        <v-col>
          <div class="d-inline-block float-right">
            <v-checkbox v-model="showHiddenIndices"
                        label="Show hidden indices"
                        class="d-inline-block mr-6 vertical-align--bottom"
                        hide-details/>
            <v-text-field id="filter"
                          v-model="filter"
                          append-icon="mdi-magnify"
                          autofocus
                          class="mt-0 pt-0 v-text-field--small"
                          hide-details
                          label="Filter..."
                          name="filter"
                          title="Filter via 'column:query'"
                          @keyup.esc="filter = ''"/>
          </div>
        </v-col>
      </v-row>
    </v-card-text>

    <v-data-table :footer-props="{itemsPerPageOptions: DEFAULT_ITEMS_PER_PAGE, showFirstLastPage: true}"
                  :headers="HEADERS"
                  :items="items"
                  :loading="loading"
                  :options.sync="options"
                  class="table--condensed table--fixed-header">
      <template v-slot:item="props">
        <index-row :index="props.item" @reloadIndices="emitReloadIndices"/>
      </template>
    </v-data-table>
  </div>
</template>

<script>
  import IndexRow from '@/components/Indices/IndexRow'
  import NewIndex from '@/components/Indices/NewIndex'
  import ElasticsearchIndex from '@/models/ElasticsearchIndex'
  import { DEFAULT_ITEMS_PER_PAGE } from '@/consts'
  import { vuexAccessors } from '@/helpers/store'
  import { ref, watch } from '@vue/composition-api'
  import { useAsyncFilter } from '@/mixins/UseAsyncTableFilter'
  import { debounce } from '@/helpers'

  export default {
    name: 'indices-table',
    components: {
      NewIndex,
      IndexRow
    },
    props: {
      indices: {
        default: () => ([]),
        type: Array
      },
      loading: {
        default: false,
        type: Boolean
      }
    },
    setup (props, context) {
      const {
        filter,
        options,
        showHiddenIndices,
        hideIndicesRegex
      } = vuexAccessors('indices', ['filter', 'options', 'showHiddenIndices', 'hideIndicesRegex'])

      const HEADERS = [
        { text: 'Name', value: 'index' },
        { text: 'Health', value: 'health' },
        { text: 'Status', value: 'status' },
        { text: 'UUID', value: 'uuid' },
        { text: 'Aliases', value: 'aliases', sortable: false },
        { text: 'Shards', value: 'parsedPri', align: 'right' },
        { text: 'Docs', value: 'parsedDocsCount', align: 'right' },
        { text: 'Storage', value: 'parsedStoreSize', align: 'right' },
        { text: '', value: 'actions', sortable: false }
      ]

      const emitReloadIndices = () => {
        context.emit('reloadIndices')
      }

      const items = ref([])
      const { asyncFilterTable } = useAsyncFilter()
      const filterTable = async () => {
        let results = props.indices
        if (!showHiddenIndices.value) {
          results = results.filter(item => !item.index.match(new RegExp(hideIndicesRegex.value)))
        }

        results = await asyncFilterTable(results, filter.value, ['index', 'uuid'])
        items.value = results.map(index => new ElasticsearchIndex(index))
      }
      const debouncedFilterTable = debounce(filterTable, 350)
      watch(filter, debouncedFilterTable)
      watch(showHiddenIndices, filterTable)
      watch(() => props.indices, filterTable)

      return {
        filter,
        options,
        showHiddenIndices,
        items,
        HEADERS,
        DEFAULT_ITEMS_PER_PAGE,
        emitReloadIndices
      }
    }
  }
</script>
