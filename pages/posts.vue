<template>
  <v-row class="fill-height" no-gutters>
    <v-col cols="12">
      <v-card>
        <v-card-title class="d-flex align-center">
          Posts
          <v-text-field
            v-model="search"
            class="ml-auto pl-2 mt-2 mt-sm-0"
            style="max-width: 500px; height: 30px;"
            dense
            solo
            single-line
            placeholder="Search by user"
          >
            <template #prepend-inner>
              <v-menu offset-y>
                <template v-slot:activator="{ on, attrs }">
                  <v-tooltip left>
                    <template #activator="{on: tooltipOn}">
                      <v-btn
                        icon
                        v-bind="attrs"
                        v-on="{...on, ...tooltipOn}"
                      >
                        <v-icon>mdi-filter-plus</v-icon>
                      </v-btn>
                    </template>
                    Add filter
                  </v-tooltip>
                </template>
                <v-list>
                  <v-list-item
                    v-for="(item, index) in filterOptions"
                    :key="index"
                    @click="selectedFilters.push(item)"
                  >
                    <v-list-item-title>{{ item.name }}</v-list-item-title>
                  </v-list-item>
                </v-list>
              </v-menu>
            </template>
          </v-text-field>
        </v-card-title>
        <v-card-title class="my-0">
          <v-row>
            <v-col v-for="(filter, i) in selectedFilters" :key="i" cols="auto">
              <selected-filter-option v-model="selectedFilters[i]" @remove="selectedFilters.splice(i, 1)" />
            </v-col>
          </v-row>
        </v-card-title>
        <v-card-text>
          <hasura-table
            source="Post"
            :fields="headers"
            :filters="allFilters"
            sort-by="created_at"
            sort-desc
            @opened="isEditItemDialogOpened = true;selectedItem = $event"
          >
            <template #[`item.edit`]="{item}">
              <v-btn
                icon
                @click="isEditItemDialogOpened = true;selectedItem = item"
              >
                <v-icon>mdi-pencil</v-icon>
              </v-btn>
            </template>
            <template #[`item.user`]="{item}">
              {{ item.user ? `${item.user.name} ${item.user.lastName}`.trim() : '[no value]' }}
            </template>
            <template #[`item.data`]="{item}">
              {{ item.data ? item.data.description : '' }}
            </template>
            <template #[`item.challengeId`]="{item}">
              {{ challenges ? (challenges.find(x => x.id === item.challengeId) || {}).name : '' }}
            </template>
            <template #[`item.content`]="{item}">
              <a v-if="item.type === 'photo'" :href="`/api/uploads/${(item.data || {}).photo}`" target="_blank">
                <img :src="`/api/uploads/w_130,h_130/${(item.data || {}).photo}`">
              </a>
              <video
                v-if="item.type === 'video'"
                :src="`/api/uploads/${(item.data || {}).video}?range=true#t=0.5`"
                width="130"
                height="130"
                style="background-color: black;"
                controls
              />
              <audio
                v-if="item.type === 'audio'"
                :src="`/api/uploads/${(item.data || {}).audio}?range=true`"
                width="130"
                controls
              />
              <span v-if="item.type === 'text'">—</span>
            </template>
            <template #[`item.comments`]="{item}">
              <v-btn
                text
                small
                @click="isEditItemDialogOpened = true;selectedItem = item"
              >
                <v-icon small class="mr-1">
                  mdi-pencil
                </v-icon>
                comments
              </v-btn>
            </template>
          </hasura-table>
        </v-card-text>
      </v-card>
    </v-col>
    <v-dialog
      v-model="isEditItemDialogOpened"
      max-width="900"
      :fullscreen="$vuetify.breakpoint.xsOnly"
      persistent
    >
      <hasura-form
        v-if="isEditItemDialogOpened"
        ref="form"
        :value="defaultItem"
        source="Post"
        :item-id="selectedItem.id"
        :fields="fields"
        @submit="onSubmit"
      >
        <template #[`title`]="{}" />
        <template #[`field.comments`]="{}">
          <h4 class="mt-3">
            Comments
          </h4>
          <hasura-table
            source="Message"
            :fields="messageHeaders"
            :filters="{postId: {_eq: selectedItem.id}}"
            sort-by="created_at"
            sort-desc
            :items-per-page="10"
            :footer-props="{ itemsPerPageOptions: [10, 50, 100] }"
          >
            <template #[`item.user`]="{item}">
              {{ item.user ? `${item.user.name} ${item.user.lastName}`.trim() : '[no value]' }}
            </template>
          </hasura-table>
        </template>
        <template #[`field.data`]="{item, $input}">
          <json-editor :value="item.data" :height="230" @input="$input('data', $event)" />
        </template>
        <template #actions="{isNew, isSaving, submit}">
          <v-card-actions>
            <v-spacer />
            <v-btn color="primary" @click="isEditItemDialogOpened = false">
              Cancel
            </v-btn>
            <v-btn color="primary" :loading="isSaving" @click="submit">
              {{ isNew ? 'Save' : 'Create' }}
            </v-btn>
          </v-card-actions>
        </template>
      </hasura-form>
    </v-dialog>
  </v-row>
</template>

<script>
import { mapState, mapActions } from 'vuex'
import { HasuraTable } from 'vuetify-hasura-table'
import { HasuraForm } from 'vuetify-hasura-form'
import { gql } from 'graphql-tag'
import JsonEditor from '~/components/JsonEditor'
import SelectedFilterOption from '~/components/SelectedFilterOption'

export default {
  transition (to, from) {
    return 'slide'
  },
  components: {
    HasuraTable,
    HasuraForm,
    JsonEditor,
    SelectedFilterOption
  },
  apollo: {
    sites: {
      query: gql`{
        sites: Site {
          id
          languages {
            isDefault
            languageId
          }
          pages {
            id
            modules (where: {systemType: {_eq: "challenges"}}) {
              id
              variables
            }
          }
        }
      }`
    }
  },
  data () {
    return {
      sites: null,
      selectedFilters: [],
      search: '',
      isEditItemDialogOpened: false,
      selectedItem: {}
    }
  },
  computed: {
    ...mapState('auth', ['user']),
    filterOptions () {
      return [
        { name: 'Challenge', value: 'challengeId', items: this.challenges, itemText: 'name', itemValue: 'id' }
      ]
    },
    defaultLanguage () {
      if (!this.sites) {
        return []
      }

      return this.sites[0].languages.find(x => x.isDefault)
    },
    challenges () {
      if (!this.sites || !this.defaultLanguage) {
        return []
      }

      return this.sites[0].pages[0].modules[0].variables.challenges.map(challenge => ({
        id: challenge.id,
        name: challenge[`name_${this.defaultLanguage.languageId}`]
      }))
    },
    defaultItem () {
      return { }
    },
    fields () {
      return [
        { label: 'Type', value: 'type', type: 'select', items: ['text', 'audio', 'video', 'audio'], md: 6 },
        { value: 'data' },
        { value: 'comments', selectable: false, editable: false }
      ]
    },
    headers () {
      return [
        { text: 'Edit', value: 'edit', sortable: false, width: 24, selectable: false },
        { text: 'Date', value: 'created_at', type: 'datetime' },
        { text: 'Challenge', value: 'challengeId', sortable: false },
        { text: 'User', value: 'user', selector: 'user {id name lastName}' },
        { text: 'Type', value: 'type' },
        { text: 'Description', value: 'data', sortable: false },
        { text: 'Content', value: 'content', sortable: false, selectable: false },
        { text: '', value: 'comments', selector: 'messages_aggregate { aggregate { count } }', sortable: false }
      ]
    },
    messageHeaders () {
      return [
        { text: 'Date', value: 'created_at', type: 'datetime' },
        { text: 'User', value: 'user', selector: 'user {id name lastName}' },
        { text: 'Text', value: 'text' }
      ]
    },
    allFilters () {
      return {
        _and: [
          Object.assign({}, ...this.selectedFilters
            .filter(filter => filter.selected)
            .map(filter => filter.resolver
              ? filter.resolver(filter.selected)
              : ({ [filter.value]: { [filter.filter || (filter.type === 'select' ? '_eq' : '_in')]: filter.selected } }))),
          this.search.length && {
            _or: [
              {
                user: {
                  _or: [
                    { name: { _ilike: `%${this.search}%` } },
                    { lastName: { _ilike: `%${this.search}%` } }
                  ]
                }
              }
            ]
          }
        ].filter(x => !!x)
      }
    }
  },
  methods: {
    ...mapActions(['refreshCache']),
    onSubmit () {
      this.isEditItemDialogOpened = false
      this.refreshCache()
    }
  },
  beforeRouteLeave (to, from, next) {
    const keepState = false
    if (!keepState) {
      this.$destroy()
    }

    next()
  }
}
</script>
