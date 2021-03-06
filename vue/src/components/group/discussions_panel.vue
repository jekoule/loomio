<script lang="coffee">
import Records            from '@/shared/services/records'
import AbilityService     from '@/shared/services/ability_service'
import EventBus           from '@/shared/services/event_bus'
import RecordLoader       from '@/shared/services/record_loader'
import ThreadFilter       from '@/shared/services/thread_filter'
import { map, debounce, orderBy, intersection, compact, omit, filter, concat, uniq } from 'lodash-es'
import Session from '@/shared/services/session'

export default
  created: ->
    @onQueryInput = debounce (val) =>
      @$router.replace(@mergeQuery(q: val))
    , 500
    @init()

  data: ->
    group: null
    discussions: []
    searchResults: []
    loader: null
    searchLoader: null
    groupIds: []

  methods:
    routeQuery: (o) ->
      @$router.replace(@mergeQuery(o))

    openStartDiscussionModal: ->
      EventBus.$emit 'openModal',
        component: 'DiscussionForm'
        props:
          discussion: Records.discussions.build
            descriptionFormat: Session.defaultFormat()
            groupId: @group.id

    beforeDestroy: ->
      EventBus.$off 'joinedGroup'

    init: ->
      Records.groups.findOrFetch(@$route.params.key).then (group) =>
        @group = group

        EventBus.$emit 'currentComponent',
          page: 'groupPage'
          title: @group.name
          group: @group
          search:
            placeholder: @$t('navbar.search_threads', name: @group.parentOrSelf().name)

        EventBus.$on 'joinedGroup', (group) => @fetch()

        @loader = new RecordLoader
          collection: 'discussions'
          params:
            group_id: @group.id
            exclude_types: 'group outcome'

        @searchLoader = new RecordLoader
          collection: 'searchResults'
          params:
            exclude_types: 'group stance outcome poll'
            subgroups: @$route.query.subgroups || 'all'
            group_id: @group.id

        @watchRecords
          key: @group.id
          collections: ['discussions', 'groups', 'memberships']
          query: (store) => @query(store)

        @refresh()

    refresh: ->
      @from = 0
      @fetch()
      @query()

    query: (store) ->
      return unless @group
      @publicGroupIds = @group.publicOrganisationIds()

      @groupIds = switch (@$route.query.subgroups || 'mine')
        when 'mine' then uniq(concat(intersection(@group.organisationIds(), Session.user().groupIds()), @publicGroupIds, @group.id)) # @group.id is present if @group is a subgroup of parentgroup that i'm a member of, and that parent group has parentMembersCanSeeDiscussions
        when 'all' then @group.organisationIds()
        else [@group.id]

      if @$route.query.q
        chain = Records.searchResults.collection.chain()
        chain = chain.find(resultGroupId: {$in: @group.parentOrSelf().organisationIds()})
        chain = chain.find(query: @$route.query.q).data()
        @searchResults = orderBy(chain, 'rank', 'desc')
      else
        chain = Records.discussions.collection.chain()
        chain = chain.find(groupId: {$in: @groupIds})

        switch @$route.query.t
          when 'unread'
            chain = chain.where (discussion) -> discussion.isUnread()
          when 'closed'
            chain = chain.find(closedAt: {$ne: null})
          when 'all'
            true # noop
          else
            chain = chain.find(closedAt: null)

        if @$route.query.tag
          chain = chain.find({tagNames: {'$contains': @$route.query.tag}})

        @discussions = chain.data()

    fetch: ->
      if @$route.query.q
        @searchLoader.fetchRecords(q: @$route.query.q)
      else
        params = {from: @from}
        params.filter = 'show_closed' if @$route.query.t == 'closed'
        params.filter = 'all' if @$route.query.t == 'all'
        params.subgroups = @$route.query.subgroups || 'mine'
        params.tags = @$route.query.tag
        @loader.fetchRecords(params)

    filterName: (filter) ->
      switch filter
        when 'unread' then 'discussions_panel.unread'
        when 'all' then 'discussions_panel.all'
        when 'closed' then 'discussions_panel.closed'
        when 'subscribed' then 'change_volume_form.simple.loud'
        else
          'discussions_panel.open'


  watch:
    '$route.params': 'init'
    '$route.query': 'refresh'

  computed:
    pinnedDiscussions: ->
      orderBy(@discussions.filter((discussion) -> discussion.pinned), ['title'], ['asc'])

    regularDiscussions: ->
      orderBy(@discussions.filter((discussion) -> !discussion.pinned), ['lastActivityAt'], ['desc'])

    groupTags: ->
      @group && @group.parentOrSelf().tagNames || []

    loading: ->
      @loader.loading || @searchLoader.loading

    noThreads: ->
      !@loading && @discussions.length == 0

    canViewPrivateContent: ->
      AbilityService.canViewPrivateContent(@group)

    canStartThread: ->
      AbilityService.canStartThread(@group)

    isLoggedIn: ->
      Session.isSignedIn()

    unreadCount: ->
      filter(@discussions, (discussion) -> discussion.isUnread()).length

</script>

<template lang="pug">
div.discussions-panel(v-if="group")
  v-layout.py-3(align-center wrap)
    //- v-select(solo hide-details flat flex-shrink :items="['Open']").mr-2
    //- v-select(solo hide-details flat flex-shrink :items="['All tags']").mr-2
    v-menu
      template(v-slot:activator="{ on }")
        v-btn.mr-2.text-lowercase.discussions-panel__filters(v-on="on" text)
          span(v-t="{path: filterName($route.query.t), args: {count: unreadCount}}")
          v-icon mdi-menu-down
      v-list(dense)
        v-list-item.discussions-panel__filters-open(@click="routeQuery({t: null})")
          v-list-item-title(v-t="'discussions_panel.open'")
        v-list-item.discussions-panel__filters-all(@click="routeQuery({t: 'all'})")
          v-list-item-title(v-t="'discussions_panel.all'")
        v-list-item.discussions-panel__filters-closed(@click="routeQuery({t: 'closed'})")
          v-list-item-title(v-t="'discussions_panel.closed'")
        v-list-item.discussions-panel__filters-unread(@click="routeQuery({t: 'unread'})")
          v-list-item-title(v-t="{path: 'discussions_panel.unread', args: { count: unreadCount }}")
        //- v-list-item(@click="routeQuery({t: 'subscribed'})")
        //-   v-list-item-title(v-t="'change_volume_form.simple.loud'")

    v-menu
      template(v-slot:activator="{ on }")
        v-btn.mr-2.text-lowercase(v-on="on" text)
          span(v-if="$route.query.tag") {{$route.query.tag}}
          span(v-else v-t="'loomio_tags.all_tags'")
          v-icon mdi-menu-down

      v-list(dense)
        v-list-item(@click="routeQuery({tag: null})" key="all")
          v-list-item-title(v-t="'loomio_tags.all_tags'")
        v-list-item(v-for="tag in groupTags" :key="tag" @click="routeQuery({tag: tag})")
          v-list-item-title {{tag}}
    v-text-field.mr-2.flex-grow-1(clearable solo hide-details :value="$route.query.q" @input="onQueryInput" :placeholder="$t('navbar.search_threads', {name: group.name})" append-icon="mdi-magnify")
    v-btn.discussions-panel__new-thread-button(@click='openStartDiscussionModal()' color='primary' v-if='canStartThread' v-t="'navbar.start_thread'")

  v-card.discussions-panel(outlined)
    div(v-if="loader.status == 403")
      p.pa-4.text-center(v-t="'error_page.forbidden'")
    div(v-else)
      .discussions-panel__content(v-if="!$route.query.q")
        .discussions-panel__list--empty.pa-4(v-if='noThreads' :value="true")
          p.text-center(v-if='canViewPrivateContent' v-t="'group_page.no_threads_here'")
          p.text-center(v-if='!canViewPrivateContent' v-t="'group_page.private_threads'")
        .discussions-panel__list.thread-preview-collection__container(v-if="discussions.length")
          v-list.thread-previews(two-line)
            thread-preview(:show-group-name="groupIds.length > 1" v-for="thread in pinnedDiscussions" :key="thread.id" :thread="thread" group-page)
            thread-preview(:show-group-name="groupIds.length > 1" v-for="thread in regularDiscussions" :key="thread.id" :thread="thread" group-page)

          v-layout(justify-center)
            v-btn.my-2.discussions-panel__show-more(outlined color='accent' v-if="!loader.exhausted" :loading="loader.loading" @click="loader.fetchRecords()" v-t="'common.action.load_more'")

          .lmo-hint-text.discussions-panel__no-more-threads.text-center.pa-1(v-t="{ path: 'group_page.no_more_threads' }", v-if='loader.numLoaded > 0 && loader.exhausted')

      .discussions-panel__content.pa-4(v-if="$route.query.q")
        p.text-center.discussions-panel__list--empty(v-if='!searchResults.length && !searchLoader.loading' v-t="{path: 'discussions_panel.no_results_found', args: {search: $route.query.q}}")
        thread-search-result(v-else v-for="result in searchResults" :key="result.id" :result="result")
</template>

<style lang="sass">
.overflow-x-auto
  overflow-x: auto
</style>
