<template>
  <table class="main">
    <thead>
      <tr>
        <th></th>
        <th v-for="field in fields" :key="field">{{field.replace(/\_/g, ' ').replace(/\|/g, ', ')}}</th>
      </tr>
    </thead>
    <tbody>
      <MergeRow
        v-for="record in records"
        :key="record.key"
        :record="record"
        :fields="fields"
        :editions="editions"
        :lists="lists"
        :bookshelves="bookshelves"
        :ratings="ratings"
        :class="{ selected: selected[record.key]}"
        :cellSelected="isCellUsed"
        :merged="merge ? merge.record : null"
        :show_diffs="show_diffs"
      >
        <template #pre>
          <td class="col-controls" v-if="['/type/edition','/type/work'].includes(record.type.key)">
            <input type="radio" name="master_key" title="Primary Record" v-model="master_key" :value="record.key" />
            <input type="checkbox" title="Include in Merge" v-model="selected[record.key]" />
          </td>
          <td v-else />
        </template>
      </MergeRow>
    </tbody>
    <tfoot>
      <MergeRow v-if="merge" :record="merge.record" :selected="selected" :fields="fields" :merged="merge">
        <template #pre>
          <td />
        </template>
      </MergeRow>
      <tr v-else><td :colspan="fields ? fields.length+1 : 1"><div>⏳</div></td></tr>
    </tfoot>
  </table>
</template>

<script>
/* eslint no-console: 0 */
import _ from 'lodash';
import Vue from 'vue';
import AsyncComputed from 'vue-async-computed';
import MergeRow from './MergeRow.vue';
import { merge, get_editions, get_lists, get_bookshelves, get_ratings } from './utils.js';

Vue.use(AsyncComputed);

/**
 * @param {string} olid
 */
function fetchRecord(olid) {
    const type = {
        W: 'works',
        M: 'books',
        A: 'authors'
    }[olid[olid.length - 1]];
    const record_key = `/${type}/${olid}`;
    // FIXME Fetch from prod openlibrary.org, otherwise it's outdated
    const url = location.host.endsWith('.openlibrary.org') ? `https://openlibrary.org${record_key}.json` : `${record_key}.json`;
    return fetch(url).then(r => {
        return (r.ok) ? r.json() : {key: record_key, type: {key: '/type/none'}, error: r.statusText};
    });
}

export default {
    name: 'MergeTable',
    components: {
        MergeRow
    },
    data() {
        return {
            master_key: null,
            /** @type {{[key: string]: Boolean}} */
            selected: []
        };
    },
    props: {
        olids: Array,
        show_diffs: Boolean
    },
    asyncComputed: {
        async records() {
            const olids_sorted = _.sortBy(this.olids, olid =>
                parseFloat(olid.match(/\d+/)[0])
            );
            // Ensure orphaned editions are at the bottom of the list
            const records = _.orderBy(
                await Promise.all(olids_sorted.map(fetchRecord)),
                record => record.type.key, 'desc');
            this.master_key = records[0].key
            this.selected = _.fromPairs(records.map(record => [record.key, record.type.key.includes('work')]));
            return records;
        },

        async editions() {
            if (!this.records) return null;

            const editionPromises = await Promise.all(
                this.records.map(r => r.type.key.includes('work') ? get_editions(r.key) : {size: 0})
            );
            const editions = editionPromises.map(p => p.value || p);
            const editionsMap = _.fromPairs(
                this.records.map((work, i) => [work.key, editions[i]])
            );

            // If any of the records are editions, insert the record as its own edition list
            Object.keys(editionsMap).forEach((key, index) => {
                if (key.includes('M')) editionsMap[key] = {size: 1, entries: [this.records[index]]};
            });

            return editionsMap;
        },

        async lists() {
            if (!this.records) return null;

            // We only need the count, so set limit=0 (waaaay faster!)
            const promises = await Promise.all(
                this.records.map(r => get_lists(r.key, 0))
            );
            const responses = promises.map(p => p.value || p);
            return _.fromPairs(
                this.records.map((work, i) => [work.key, responses[i]])
            );
        },
        async bookshelves() {
            if (!this.records) return null;

            const promises = await Promise.all(
                this.records.map(r => get_bookshelves(r.key))
            );
            const responses = promises.map(p => p.value || p);
            return _.fromPairs(
                this.records.map((work, i) => [work.key, responses[i].counts])
            );
        },

        async ratings() {
            if (!this.records) return null;

            const promises = await Promise.all(
                this.records.map(r => get_ratings(r.key))
            );
            const responses = promises.map(p => p.value || p);
            return _.fromPairs(
                this.records.map((work, i) => [work.key, responses[i]])
            );
        },

        async merge() {
            if (!this.master_key || !this.records || !this.editions || !this.lists || !this.bookshelves)
                return undefined;

            const master = this.records.find(r => r.key === this.master_key);
            const dupes = this.records
                .filter(r => this.selected[r.key])
                .filter(r => r.key !== this.master_key);
            const work_dupes = dupes.filter(r => r.type.key === '/type/work');
            const records = [master, ...dupes];
            const editions_to_move = _.flatMap(
                dupes,
                work => this.editions[work.key].entries
            );

            const [record, sources] = merge(master, work_dupes);

            const extras = {
                edition_count: _.sum(records.map(r => this.editions[r.key].size)),
                list_count: _.sum(records.map(r => this.lists[r.key].size))
            };

            return { record, sources, ...extras, dupes, editions_to_move };
        }
    },
    methods: {
        isCellUsed(record, field) {
            if (!this.merge) return false;
            return field in this.merge.sources
                ? this.merge.sources[field].includes(record.key)
                : record.key === this.master_key;
        }
    },
    computed: {
        fields() {
            const at_start = ['covers'];
            const together = ['key', 'title', 'subtitle', 'authors', 'error'];
            const subjects = [
                'subjects',
                'subject_people',
                'subject_places',
                'subject_times'
            ];
            const record_data = [
                'created',
                'last_modified',
                'revision',
                'type',
                'location'
            ];
            const identifiers = [
                'first_publish_date',
                'dewey_number',
                'lc_classifications'
            ];
            const text_data = [
                'description',
                'excerpts',
                'first_sentence',
                'links'
            ];
            const exclude = [
                'latest_revision',
                'id',
            ];
            const recordFields = _.uniq(_.flatMap(this.records, Object.keys));
            const otherFields = _.difference(recordFields, [
                ...at_start,
                ...together,
                ...subjects,
                ...record_data,
                ...identifiers,
                ...text_data,
                ...exclude
            ]);
            const usedIdentifiers = _.intersection(identifiers, recordFields);
            const usedTextData = _.intersection(text_data, recordFields);
            return [
                ...at_start,
                together.join('|'),
                record_data.join('|'),
                'editions',
                'references',
                ...usedIdentifiers,
                subjects.join('|'),
                ...usedTextData,
                ...otherFields
            ];
        }
    }
};
</script>

<style lang="less">
@row-height: 105px;
@row-padding: 8px;
@table-background: rgb(248, 248, 248);
@selection-background: rgb(220, 224, 238);

body {
  font-size: .85em;
}
time {
  white-space: nowrap;
}

table.main {
  border-collapse: collapse;
  min-width: 100%;

  thead,
  tfoot {
    position: sticky;
    z-index: 300;
  }
  & > thead {
    top: 0;
    & > tr > th {
      font-variant: small-caps;
      margin-right: 4px;
      background: rgb(240, 237, 226);
    }
  }
  & > tbody {
    background: @table-background;
  }
  & > tfoot {
    background: @selection-background;
    bottom: 0;

    & > tr {
      border-top: 4px double;
      box-shadow: 0 2px 4px inset black;

      & > td > div {
        min-height: @row-height;
      }
    }
  }

  & > tbody > tr,
  & > tfoot > tr {
    &:hover {
      background: rgba(200, 200, 0, .1);
    }
    & > td {
      max-height: @row-height;
      max-width: 300px;
      position: relative;
      vertical-align: top;
      border-bottom: 4px solid rgba(255, 255, 255, 0.9);
      box-sizing: border-box;
      padding: 0;

      & > div {
        height: calc(@row-height - @row-padding * 2);
        max-height: calc(@row-height - @row-padding * 2);
        overflow-y: auto;
        padding: @row-padding;
        margin-right: 4px;
      }

      & > div.field-covers {
        width: 100px;
        overflow-y: auto;
        overflow-x: hidden;
        float: left;
        margin-right: 8px;
        .wrapper {
          img {
            width: 100%;
          }
        }
      }

      & > div.wrap-key--title--subtitle--authors--error {
        min-width: max-content;
        padding: 0 0 calc(@row-padding * 2) 0;

        & > div {
          padding: @row-padding @row-padding 0 @row-padding;
          &:last-child {
            padding-bottom: @row-padding;
          }
        }

        div.field-subtitle {
          padding-left: 1em;
          padding-top: 0;
        }
      }

      & > div.wrap-created--last_modified--revision--type--location {
        white-space: nowrap;
        padding: @row-padding 0 @row-padding 0;

        & > div {
          padding: calc(@row-padding/2) @row-padding 0 @row-padding;
          &:nth-child(1), &:nth-child(2) {
            padding-top: @row-padding;
          }
          &:last-child {
            padding-bottom: @row-padding;
          }
        }

        & > div.field-created { padding-right: 0; }
        & > div.field-last_modified { padding-left: 0; }
        & > div.field-created, & > div.field-last_modified {
          display: inline;
          font-size: 0.95em;
        }
        & > div.field-last_modified::before { content: "…"; }
        & > div.field-revision > div::before { content: "v"; }

      }

      & > .td-container {
        overflow-y: auto;
        resize: vertical;
      }
    }

    & > td.col-key--title--subtitle--authors--error,
    & > .col-subjects--subject_people--subject_places--subject_times,
    & > td.col-editions  {
      max-width: max-content;
    }
  }

  & > tbody .work:not(.selected) {
    opacity: .5;
  }
}

.field-container {
  overflow-y: auto;
}

.field-container.selected {
  background: @selection-background;
}

td.col-controls {
  text-align: center;
  line-height: 2em;
}

div.field-error {
  font-weight: 700;
  color: red;
}

td.col-description {
  min-width: 200px;
}

td.col-description, td.col-excerpts, td.col-first_sentence, td.col-links {
  font-size: .9em;
  div {
    max-height: @row-height;
  }
  ul {
    padding: 0;
    margin: 0;
  }
}

div.field-lc_classifications li, div.field-dewey_number li {
  font-size: .9em;
  white-space: nowrap;
}

li.excerpt-item {
  padding-bottom: 0.4em;
}

.col-subjects--subject_people--subject_places--subject_times >
  div.wrap-subjects--subject_people--subject_places--subject_times {
  height: @row-height;
  max-height: @row-height;
  display: flex;
  flex-direction: column;
  padding: 0;
  .field-container {
    min-height: 16px;
    padding: 4px;
    border-bottom: 2px solid @table-background;
    flex: 1 1 auto;
    &:last-child{
        border-bottom: 0;
    }
  }
}

.field-authors {
  thead, td.author-index, td.author-type {
    display: none;
  }
}

ul.reset {
  padding: 0;
  margin: 0;
  & > li {
    list-style: none;
  }
}

.pill-list {
  min-width: 250px;
  text-align: center;
}
.pill {
  font-size: .9em;
  display: inline-block;
  white-space: nowrap;
  border: 1px solid;
  border-radius: 10px;
  padding: 0 6px;
  text-align: center;
  margin: 0 2px 1px 0;
  background: rgba(255, 255, 255, .4);
}

.field-subject_people::before {
  content: "👤";
  float: left;
  padding-left: 3px;
}
.field-subject_places::before {
  content: "🌎";
  float: left;
  padding-left: 3px;
}
.field-subject_times::before {
  content: "🕗";
  float: left;
  padding-left: 3px;
}

td.col-editions div.td-container {
  width: 400px;
  max-height: calc(@row-height - 30px);
}

.col-references > div {
  white-space: nowrap;
  & > div {
    padding-bottom: @row-padding;
  }
}

div.field-links li {
  margin-bottom: 8px;
}

.bookshelf-counts span:not(:first-child)::before {
  content: " / ";
}
</style>
