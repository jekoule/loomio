<script lang="coffee">
import Records from '@/shared/services/records'
import RecordLoader from '@/shared/services/record_loader'
import WatchRecords from '@/mixins/watch_records'
import { debounce, some, every } from 'lodash'

export default
  mixins: [WatchRecords]

  data: ->
    group: Records.groups.fuzzyFind(@$route.params.key)
    polls: []
    fragment: ''
    loader: null
    per: 25
    from: 0
    includeSubgroups: false

  created: ->
    @loader = new RecordLoader
      collection: 'polls'
      path: 'search'
      params:
        group_key: @group.key
        per: @per

    @fetch()

    @watchRecords
      collections: ['polls']
      query: (store) =>
        chain = store.polls.collection.chain()

        if @includeSubgroups
          chain = chain.find(groupId: {$in: @group.organisationIds()})
        else
          chain = chain.find(groupId: @group.id)


        if @fragment
          chain = chain.where (poll) =>
            some [poll.title, poll.details], (field) =>
              every @fragment.split(' '), (frag) -> RegExp(frag, "i").test(field)

        @polls = chain.simplesort('closing_at').limit(@loader.numRequested).data()

  methods:
    fetch: debounce ->
      @loader.fetchRecords
        from: @from
        query: @fragment
        include_subgroups: @includeSubgroups
    ,
      300

  computed:
    totalRecords: -> @group.pollsCount
    showLoadMore: ->
      !@loader.loading && @loader.numRequested < @totalRecords && !@fragment.length
  watch:
    fragment: -> @fetch()
    includeSubgroups: -> @fetch()

</script>

<template lang="pug">
div
  v-toolbar(flat)
    v-toolbar-items
      v-text-field(solo flat v-model="fragment" append-icon="mdi-magnify" :label="$t('common.action.search')" clearable)
    v-spacer
    v-switch(v-if="group.hasSubgroups()" v-model="includeSubgroups" :label="$t('discussions_panel.include_subgroups')")
    v-progress-linear(color="accent" indeterminate :active="loader.loading" absolute bottom)

  .group-polls-panel
    v-list(two-line avatar v-if='polls.length')
      poll-common-preview(:poll='poll', v-for='poll in polls', :key='poll.id')

  v-alert(v-if='polls.length == 0 && !loader.loading' :value="true" color="grey" outlined icon="info" v-t="'group_polls_panel.no_polls'")

    //- | Showing x of {{totalRecords}} totalj
  v-layout(align-center)
    span(v-if="!includeSubgroups && !fragment" v-t="{path: 'members_panel.loaded_of_total', args: {loaded: loader.numLoaded, total: totalRecords}}")
    v-btn(v-if="showLoadMore" :loading="loader.loading" @click="loader.loadMore()" v-t="'common.action.load_more'")
</template>