<template>
  <div>
    <toolbar-sticky v-if="showToolbar">
      <v-toolbar-title>
        <span>{{ $t('search.search_results_for', {name: $route.query.q}) }}</span>
      </v-toolbar-title>
    </toolbar-sticky>

    <multi-select-bar
      v-model="selectedSeries"
      kind="series"
      @unselect-all="selectedSeries = []"
      @mark-read="markSelectedSeriesRead"
      @mark-unread="markSelectedSeriesUnread"
      @add-to-collection="addToCollection"
      @edit="editMultipleSeries"
    />

    <multi-select-bar
      v-model="selectedBooks"
      kind="books"
      @unselect-all="selectedBooks = []"
      @mark-read="markSelectedBooksRead"
      @mark-unread="markSelectedBooksUnread"
      @add-to-readlist="addToReadList"
      @edit="editMultipleBooks"
      @bulk-edit="bulkEditMultipleBooks"
    />

    <multi-select-bar
      v-model="selectedCollections"
      kind="collections"
      @unselect-all="selectedCollections = []"
      @delete="deleteCollections"
    />

    <multi-select-bar
      v-model="selectedReadLists"
      kind="readlists"
      @unselect-all="selectedReadLists = []"
      @delete="deleteReadLists"
    />

    <v-container fluid>
      <empty-state
        v-if="emptyResults"
        :title="$t('search.no_results')"
        :sub-title="$t('search.search_for_something_else')"
        icon="mdi-magnify"
        icon-color="secondary"
        class="my-4"
      >
      </empty-state>

      <template v-else>
        <horizontal-scroller
          v-if="loaderSeries && loaderSeries.items.length !== 0"
          class="mb-4"
          :tick="loaderSeries.tick"
          @scroll-changed="(percent) => scrollChanged(loaderSeries, percent)"
        >
          <template v-slot:prepend>
            <div class="title">{{ $t('common.series') }}</div>
          </template>
          <template v-slot:content>
            <item-browser :items="loaderSeries.items"
                          nowrap
                          :edit-function="isAdmin ? singleEditSeries : undefined"
                          :selected.sync="selectedSeries"
                          :selectable="selectedBooks.length === 0 && selectedCollections.length === 0 && selectedReadLists.length === 0"
                          :fixed-item-width="fixedCardWidth"
            />
          </template>
        </horizontal-scroller>

        <horizontal-scroller
          v-if="loaderBooks && loaderBooks.items.length !== 0"
          class="mb-4"
          :tick="loaderBooks.tick"
          @scroll-changed="(percent) => scrollChanged(loaderBooks, percent)"
        >
          <template v-slot:prepend>
            <div class="title">{{ $t('common.books') }}</div>
          </template>
          <template v-slot:content>
            <item-browser :items="loaderBooks.items"
                          :item-context="[ItemContext.SHOW_SERIES]"
                          nowrap
                          :edit-function="isAdmin ? singleEditBook : undefined"
                          :selected.sync="selectedBooks"
                          :selectable="selectedSeries.length === 0 && selectedCollections.length === 0 && selectedReadLists.length === 0"
                          :fixed-item-width="fixedCardWidth"
            />
          </template>
        </horizontal-scroller>

        <horizontal-scroller
          v-if="loaderCollections && loaderCollections.items.length !== 0"
          class="mb-4"
          :tick="loaderCollections.tick"
          @scroll-changed="(percent) => scrollChanged(loaderCollections, percent)"
        >
          <template v-slot:prepend>
            <div class="title">{{ $t('common.collections') }}</div>
          </template>
          <template v-slot:content>
            <item-browser :items="loaderCollections.items"
                          nowrap
                          :edit-function="isAdmin ? singleEditCollection : undefined"
                          :selected.sync="selectedCollections"
                          :selectable="isAdmin && selectedSeries.length === 0 && selectedBooks.length === 0 && selectedReadLists.length === 0"
                          :fixed-item-width="fixedCardWidth"
            />
          </template>
        </horizontal-scroller>

        <horizontal-scroller
          v-if="loaderReadLists && loaderReadLists.items.length !== 0"
          class="mb-4"
          :tick="loaderReadLists.tick"
          @scroll-changed="(percent) => scrollChanged(loaderReadLists, percent)"
        >
          <template v-slot:prepend>
            <div class="title">{{ $t('common.readlists') }}</div>
          </template>
          <template v-slot:content>
            <item-browser :items="loaderReadLists.items"
                          nowrap
                          :edit-function="isAdmin ? singleEditReadList : undefined"
                          :selected.sync="selectedReadLists"
                          :selectable="isAdmin && selectedSeries.length === 0 && selectedBooks.length === 0 && selectedCollections.length === 0"
                          :fixed-item-width="fixedCardWidth"
            />
          </template>
        </horizontal-scroller>

      </template>
    </v-container>

  </div>
</template>

<script lang="ts">
import MultiSelectBar from '@/components/bars/MultiSelectBar.vue'
import ToolbarSticky from '@/components/bars/ToolbarSticky.vue'
import EmptyState from '@/components/EmptyState.vue'
import HorizontalScroller from '@/components/HorizontalScroller.vue'
import ItemBrowser from '@/components/ItemBrowser.vue'
import {BookDto} from '@/types/komga-books'
import {
  BOOK_CHANGED,
  BOOK_DELETED,
  COLLECTION_CHANGED,
  COLLECTION_DELETED,
  LIBRARY_DELETED,
  READLIST_CHANGED,
  READLIST_DELETED,
  READPROGRESS_CHANGED,
  READPROGRESS_DELETED,
  READPROGRESS_SERIES_CHANGED,
  READPROGRESS_SERIES_DELETED,
  SERIES_CHANGED,
  SERIES_DELETED,
} from '@/types/events'
import Vue from 'vue'
import {SeriesDto} from '@/types/komga-series'
import {
  BookSseDto,
  CollectionSseDto,
  ReadListSseDto,
  ReadProgressSeriesSseDto,
  ReadProgressSseDto,
  SeriesSseDto,
} from '@/types/komga-sse'
import {throttle} from 'lodash'
import {PageLoader} from '@/types/pageLoader'
import {ItemContext} from '@/types/items'

export default Vue.extend({
  name: 'SearchView',
  components: {
    EmptyState,
    ToolbarSticky,
    HorizontalScroller,
    ItemBrowser,
    MultiSelectBar,
  },
  data: () => {
    return {
      ItemContext,
      loaderSeries: undefined as unknown as PageLoader<SeriesDto>,
      loaderBooks: undefined as unknown as PageLoader<BookDto>,
      loaderCollections: undefined as unknown as PageLoader<CollectionDto>,
      loaderReadLists: undefined as unknown as PageLoader<ReadListDto>,
      pageSize: 20,
      loading: false,
      selectedSeries: [] as SeriesDto[],
      selectedBooks: [] as BookDto[],
      selectedCollections: [] as CollectionDto[],
      selectedReadLists: [] as ReadListDto[],
    }
  },
  created() {
    this.$eventHub.$on(LIBRARY_DELETED, this.reloadResults)
    this.$eventHub.$on(SERIES_CHANGED, this.seriesChanged)
    this.$eventHub.$on(SERIES_DELETED, this.seriesChanged)
    this.$eventHub.$on(BOOK_CHANGED, this.bookChanged)
    this.$eventHub.$on(BOOK_DELETED, this.bookChanged)
    this.$eventHub.$on(COLLECTION_CHANGED, this.collectionChanged)
    this.$eventHub.$on(COLLECTION_DELETED, this.collectionChanged)
    this.$eventHub.$on(READLIST_CHANGED, this.readListChanged)
    this.$eventHub.$on(READLIST_DELETED, this.readListChanged)
    this.$eventHub.$on(READPROGRESS_CHANGED, this.readProgressChanged)
    this.$eventHub.$on(READPROGRESS_DELETED, this.readProgressChanged)
    this.$eventHub.$on(READPROGRESS_SERIES_CHANGED, this.readProgressSeriesChanged)
    this.$eventHub.$on(READPROGRESS_SERIES_DELETED, this.readProgressSeriesChanged)
  },
  beforeDestroy() {
    this.$eventHub.$off(LIBRARY_DELETED, this.reloadResults)
    this.$eventHub.$off(SERIES_CHANGED, this.seriesChanged)
    this.$eventHub.$off(SERIES_DELETED, this.seriesChanged)
    this.$eventHub.$off(BOOK_CHANGED, this.bookChanged)
    this.$eventHub.$off(BOOK_DELETED, this.bookChanged)
    this.$eventHub.$off(COLLECTION_CHANGED, this.collectionChanged)
    this.$eventHub.$off(COLLECTION_DELETED, this.collectionChanged)
    this.$eventHub.$off(READLIST_CHANGED, this.readListChanged)
    this.$eventHub.$off(READLIST_DELETED, this.readListChanged)
    this.$eventHub.$off(READPROGRESS_CHANGED, this.readProgressChanged)
    this.$eventHub.$off(READPROGRESS_DELETED, this.readProgressChanged)
    this.$eventHub.$off(READPROGRESS_SERIES_CHANGED, this.readProgressSeriesChanged)
    this.$eventHub.$off(READPROGRESS_SERIES_DELETED, this.readProgressSeriesChanged)
  },
  watch: {
    '$route.query.q': {
      handler: function (val) {
        this.setupLoaders(val)
        this.loadResults(val)
        this.selectedBooks = []
        this.selectedSeries = []
        this.selectedCollections = []
        this.selectedReadLists = []
      },
      deep: true,
      immediate: true,
    },
  },
  computed: {
    isAdmin(): boolean {
      return this.$store.getters.meAdmin
    },
    fixedCardWidth(): number {
      return this.$vuetify.breakpoint.xs ? 120 : 150
    },
    showToolbar(): boolean {
      return this.selectedSeries.length === 0 && this.selectedBooks.length === 0 && this.selectedCollections.length === 0 && this.selectedReadLists.length === 0
    },
    emptyResults(): boolean {
      return !this.loading &&
        this.loaderSeries?.items.length === 0 &&
        this.loaderBooks?.items.length === 0 &&
        this.loaderCollections?.items.length === 0 &&
        this.loaderReadLists?.items.length === 0
    },
  },
  methods: {
    async scrollChanged(loader: PageLoader<any>, percent: number) {
      if (percent > 0.95) await loader.loadNext()
    },
    seriesChanged(event: SeriesSseDto) {
      if (this.loaderSeries?.items.some(x => x.id === event.seriesId)) {
        this.reloadResults()
      }
    },
    bookChanged(event: BookSseDto) {
      if (this.loaderBooks?.items.some(x => x.id === event.bookId)) {
        this.reloadResults()
      }
    },
    readProgressChanged(event: ReadProgressSseDto) {
      if (this.loaderBooks?.items.some(x => x.id === event.bookId)) {
        this.reloadResults()
      }
    },
    readProgressSeriesChanged(event: ReadProgressSeriesSseDto) {
      if (this.loaderSeries?.items.some(x => x.id === event.seriesId)) {
        this.reloadResults()
      }
    },
    collectionChanged(event: CollectionSseDto) {
      if (this.loaderCollections?.items.some(x => x.id === event.collectionId)) {
        this.reloadResults()
      }
    },
    readListChanged(event: ReadListSseDto) {
      if (this.loaderReadLists?.items.some(x => x.id === event.readListId)) {
        this.reloadResults()
      }
    },
    singleEditSeries(series: SeriesDto) {
      this.$store.dispatch('dialogUpdateSeries', series)
    },
    singleEditBook(book: BookDto) {
      this.$store.dispatch('dialogUpdateBooks', book)
    },
    singleEditCollection(collection: CollectionDto) {
      this.$store.dispatch('dialogEditCollection', collection)
    },
    singleEditReadList(readList: ReadListDto) {
      this.$store.dispatch('dialogEditReadList', readList)
    },
    async markSelectedSeriesRead() {
      await Promise.all(this.selectedSeries.map(s =>
        this.$komgaSeries.markAsRead(s.id),
      ))
      this.selectedSeries = await Promise.all(this.selectedSeries.map(s =>
        this.$komgaSeries.getOneSeries(s.id),
      ))
    },
    async markSelectedSeriesUnread() {
      await Promise.all(this.selectedSeries.map(s =>
        this.$komgaSeries.markAsUnread(s.id),
      ))
      this.selectedSeries = await Promise.all(this.selectedSeries.map(s =>
        this.$komgaSeries.getOneSeries(s.id),
      ))
    },
    addToCollection() {
      this.$store.dispatch('dialogAddSeriesToCollection', this.selectedSeries)
    },
    addToReadList() {
      this.$store.dispatch('dialogAddBooksToReadList', this.selectedBooks)
    },
    editMultipleSeries() {
      this.$store.dispatch('dialogUpdateSeries', this.selectedSeries)
    },
    editMultipleBooks() {
      this.$store.dispatch('dialogUpdateBooks', this.selectedBooks)
    },
    bulkEditMultipleBooks() {
      this.$store.dispatch('dialogUpdateBulkBooks', this.selectedBooks)
    },
    deleteCollections() {
      this.$store.dispatch('dialogDeleteCollection', this.selectedCollections)
    },
    deleteReadLists() {
      this.$store.dispatch('dialogDeleteReadList', this.selectedReadLists)
    },
    async markSelectedBooksRead() {
      await Promise.all(this.selectedBooks.map(b =>
        this.$komgaBooks.updateReadProgress(b.id, {completed: true}),
      ))
    },
    async markSelectedBooksUnread() {
      await Promise.all(this.selectedBooks.map(b =>
        this.$komgaBooks.deleteReadProgress(b.id),
      ))
    },
    reloadResults: throttle(function (this: any) {
      this.loadResults(this.$route.query.q.toString(), true)
    }, 500),
    setupLoaders(search: string) {
      if (search) {
        this.loaderSeries = new PageLoader<SeriesDto>({size: this.pageSize}, (pageable: PageRequest) => this.$komgaSeries.getSeries(undefined, pageable, search))
        this.loaderBooks = new PageLoader<BookDto>({size: this.pageSize}, (pageable: PageRequest) => this.$komgaBooks.getBooks(undefined, pageable, search))
        this.loaderCollections = new PageLoader<CollectionDto>({size: this.pageSize}, (pageable: PageRequest) => this.$komgaCollections.getCollections(undefined, pageable, search))
        this.loaderReadLists = new PageLoader<ReadListDto>({size: this.pageSize}, (pageable: PageRequest) => this.$komgaReadLists.getReadLists(undefined, pageable, search))
      } else {
        this.loaderSeries = null as unknown as PageLoader<SeriesDto>
        this.loaderBooks = null as unknown as PageLoader<BookDto>
        this.loaderCollections = null as unknown as PageLoader<CollectionDto>
        this.loaderReadLists = null as unknown as PageLoader<ReadListDto>
      }
    },
    async loadResults(search: string, reload: boolean = false) {
      this.selectedBooks = []
      this.selectedSeries = []
      this.selectedCollections = []
      this.selectedReadLists = []

      if (search) {
        this.loading = true
        if (reload) {
          Promise.all([
            this.loaderSeries.reload(),
            this.loaderBooks.reload(),
            this.loaderCollections.reload(),
            this.loaderReadLists.reload(),
          ]).then(() => {
            this.loading = false
          })
        } else {
          Promise.all([
            this.loaderSeries.loadNext(),
            this.loaderBooks.loadNext(),
            this.loaderCollections.loadNext(),
            this.loaderReadLists.loadNext(),
          ]).then(() => {
            this.loading = false
          })
        }
      }
    },
  },
})
</script>
<style scoped>
</style>
