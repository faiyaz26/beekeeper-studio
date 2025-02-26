<template>
  <div
    class="sidebar detail-view-sidebar flex-col"
    ref="sidebar"
    v-show="!hidden"
  >
    <div class="header">
      <div class="header-group">
        <span class="title sub">{{ sidebarTitle }}</span>
        <button
          class="close-btn btn btn-fab"
          @click="close"
        >
          <i class="material-icons">close</i>
        </button>
      </div>
      <div
        class="header-group"
        v-show="!empty"
      >
        <div class="filter-wrap">
          <input
            class="form-control"
            type="text"
            placeholder="Filter fields"
            v-model="debouncedFilter"
          >
          <button
            type="button"
            class="clear btn-link"
            @click="filter = ''"
            v-if="filter"
          >
            <i class="material-icons">cancel</i>
          </button>
        </div>
      </div>
    </div>
    
    <div class="json-table-view" v-if="!empty">
      <table v-if="hasFilteredData">
        <thead>
          <tr>
            <th class="key-cell">Field Name</th>
            <th class="value-cell">Field Value</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(value, key) in filteredJson" :key="key">
            <td class="key-cell">{{ key }}</td>
            <td class="value-cell">
              <div class="value-wrapper">
                <span v-if="isExpandable(key)" class="expandable-value" @click="expandPath(findExpandablePath(key))">
                  {{ value }}
                  <i class="material-icons">chevron_right</i>
                </span>
                <span v-else-if="isTruncated(key)" class="truncated-value" @click="restoreTruncatedPath(key)">
                  {{ value }}
                  <i class="material-icons">more_horiz</i>
                </span>
                <span v-else>{{ value }}</span>
                <button class="copy-btn" @click="copyValue(value)" title="Copy value">
                  <i class="material-icons">content_copy</i>
                </button>
              </div>
            </td>
          </tr>
        </tbody>
      </table>
      <div v-else class="no-results">
        No matching data found
      </div>
    </div>
    
    <div class="empty-state" v-show="empty">
      No Data
    </div>
  </div>
</template>

<script lang="ts">
/**
 * hidden:  it's recommended to use `hidden` prop instead of v-show so that
 *          the text editor can be reinitialized.
 * dataId:  use this to update the component with new data.
 */
import Vue from "vue";
import TextEditor from "@/components/common/texteditor/TextEditor.vue";
import {
  ExpandablePath,
  findKeyPosition,
  findValueInfo,
  createExpandableElement,
  createTruncatableElement,
  deepFilterObjectProps,
  getPaths,
  eachPaths,
} from "@/lib/data/detail_view";
import { mapGetters } from "vuex";
import { EditorMarker } from "@/lib/editor/utils";
import { persistJsonFold } from "@/lib/editor/plugins/persistJsonFold";
import DetailViewSidebarUpsell from '@/components/upsell/DetailViewSidebarUpsell.vue'
import rawLog from "@bksLogger";
import _ from "lodash";
import globals from '@/common/globals'

const log = rawLog.scope("detail-view-sidebar");

export default Vue.extend({
  components: { TextEditor, DetailViewSidebarUpsell },
  props: ["value", "hidden", "expandablePaths", "dataId", "title", "reinitialize"],
  data() {
    return {
      reinitializeTextEditor: 0,
      filter: "",
      foldAll: 0,
      unfoldAll: 0,
      restoredTruncatedPaths: [],
    };
  },
  watch: {
    hidden() {
      if (!this.hidden) this.reinitializeTextEditor++;
    },
    dataId() {
      if (this.expandFKDetailsByDefault) {
        this.expandablePaths.forEach((expandablePath: ExpandablePath) => {
          // Expand only the first level
          if (expandablePath.path.length === 1) {
            this.expandPath(expandablePath);
          }
        });
      }
    },
  },
  computed: {
    sidebarTitle() {
      return this.title ?? "JSON Row Tester"
    },
    empty() {
      return _.isEmpty(this.value);
    },
    mode() {
      if (!this.value) {
        return null;
      }
      return { name: "javascript", json: true };
    },
    text() {
      if (this.empty) {
        return "";
      }
      if (this.filter) {
        const filtered = deepFilterObjectProps(this.processedValue, this.filter);
        return JSON.stringify(filtered, null, 2);
      }
      return JSON.stringify(this.processedValue, null, 2);
    },
    debouncedFilter: {
      get() {
        return this.filter;
      },
      set: _.debounce(function (value) {
        this.filter = value;
      }, 500),
    },
    processedValue() {
      const clonedValue = _.cloneDeep(this.value)
      eachPaths(clonedValue, (path, value) => {
        if (this.truncatedPaths.includes(path)) {
          _.set(clonedValue, path, (value as string).slice(0, globals.maxDetailViewTextLength))
        }
      })
      return clonedValue
    },
    truncatablePaths() {
      return getPaths(this.value).filter((path) => {
        const val = _.get(this.value, path)
        if (
          typeof val === "string" &&
          val.length > globals.maxDetailViewTextLength
        ) {
          return true
        }
        return false
      })
    },
    truncatedPaths() {
      return _.difference(this.truncatablePaths, this.restoredTruncatedPaths)
    },
    markers() {
      const markers: EditorMarker[] = [];
      _.forEach(this.expandablePaths, (expandablePath: ExpandablePath) => {
        try {
          const line = findKeyPosition(this.text, expandablePath.path);
          const { from, to, value } = findValueInfo(this.lines[line]);
          const element = createExpandableElement(value);
          const onClick = (_event) => {
            this.expandPath(expandablePath);
          };
          markers.push({
            type: "custom",
            from: { line, ch: from },
            to: { line, ch: to },
            onClick,
            element,
          });
        } catch (e) {
          log.warn("Failed to mark expandable path", expandablePath);
          log.warn(e);
        }
      });
      _.forEach(this.truncatedPaths, (path) => {
        // Avoid conflicts with expandable paths
        if (this.expandablePaths.includes(path)) {
          return
        }
        try {
          const line = findKeyPosition(this.text, path.split("."));
          const { from, to, value } = findValueInfo(this.lines[line]);
          const element = createTruncatableElement(value);
          const onClick = async () => {
            this.restoredTruncatedPaths.push(path)
            await this.$nextTick()
            this.reinitializeTextEditor++
          }
          markers.push({
            type: "custom",
            from: { line, ch: from },
            to: { line, ch: to },
            onClick,
            element,
          });
        } catch (e) {
          log.warn("Failed to mark truncated path", path);
          log.warn(e);
        }
      })
      return markers;
    },
    lines() {
      return this.text?.split("\n") || [];
    },
    menuOptions() {
      return [
        {
          name: "Copy Visible",
          handler: () => {
            this.$native.clipboard.writeText(this.text);
          },
        },
        {
          name: "Collapse all",
          handler: () => {
            this.foldAll++;
          },
        },
        {
          name: "Expand all",
          handler: () => {
            this.unfoldAll++;
          },
        },
        {
          name: "Always Expand Foreign Keys",
          handler: () => {
            this.$store.dispatch("toggleExpandFKDetailsByDefault");
          },
          checked: this.expandFKDetailsByDefault,
        },

      ]
    },
    textEditorPlugins() {
      return [persistJsonFold]
    },
    flattenedJson() {
      if (this.empty) return {};
      
      const result = {};
      const flatten = (obj, prefix = '') => {
        for (const key in obj) {
          const value = obj[key];
          const newKey = prefix ? `${prefix}.${key}` : key;
          
          if (typeof value === 'object' && value !== null) {
            // For objects, show a preview
            result[newKey] = '{...}';
          } else {
            result[newKey] = value;
          }
        }
      };
      
      flatten(this.processedValue);
      return result;
    },
    filteredJson() {
      if (!this.filter) return this.flattenedJson;
      
      const filterLower = this.filter.toLowerCase();
      return Object.entries(this.flattenedJson).reduce((acc, [key, value]) => {
        if (key.toLowerCase().includes(filterLower) || 
            String(value).toLowerCase().includes(filterLower)) {
          acc[key] = value;
        }
        return acc;
      }, {});
    },
    hasFilteredData() {
      return Object.keys(this.filteredJson).length > 0;
    },
    ...mapGetters(["expandFKDetailsByDefault"]),
  },
  methods: {
    expandPath(path: ExpandablePath) {
      this.$emit("expandPath", path);
    },
    close() {
      this.$emit("close")
    },
    isExpandable(key) {
      return this.expandablePaths.some(path => 
        path.path.join('.') === key
      );
    },

    findExpandablePath(key) {
      return this.expandablePaths.find(path => 
        path.path.join('.') === key
      );
    },

    isTruncated(key) {
      return this.truncatedPaths.includes(key);
    },

    restoreTruncatedPath(path) {
      this.restoredTruncatedPaths.push(path);
    },

    copyValue(value) {
      this.$native.clipboard.writeText(String(value));
    },
  }
});
</script>
