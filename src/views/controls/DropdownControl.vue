<template>
  <div class="dropdown-control-feild">
    <select
      :id="control.uniqueId"
      v-show="!control.multiple"
      :class="controlFieldClass"
      :name="control.name || control.uniqueId"
      @change="onChange($event, control.multiple)"
      :key="listOptions.length"
    >
      <!-- placeholder -->
      <option
        disabled
        :selected="!value || !value.length || !listOptions.length"
        v-text="control.placeholderText"
        v-if="control.placeholderText"
      ></option>
      <option disabled v-if="!listOptions.length">Getting Data...</option>

      <!-- list rendering -->
      <option
        v-for="optionObj in listOptions"
        :key="optionObj.value"
        :value="optionObj.value"
        v-text="optionObj.text"
        :selected="
          `${value}` == `${optionObj.value}` ||
          (Array.isArray(value) &&
            value.map((item) => `${item}`).indexOf(`${optionObj.value}`) > -1)
            ? 'selected'
            : ''
        "
      ></option>
    </select>
    <div
      v-show="control.multiple"
      :class="controlFieldClass"
      class="multi-select"
      :key="`${listOptions.length}_select`"
    >
      <div class="option" v-if="!listOptions.length">Getting Data...</div>
      <div
        v-for="optionObj in listOptions"
        :key="`${optionObj.value}_option`"
        @click="optionClicked(optionObj.value)"
        :class="{
          selected:
            `${selectedItems}` == `${optionObj.value}` ||
            (Array.isArray(selectedItems) &&
              selectedItems.map((item) => `${item}`).indexOf(`${optionObj.value}`) >
                -1),
        }"
        class="option"
      >
        {{ optionObj.text }}
      </div>
    </div>
  </div>
</template>

<script>
import { CONTROL_FIELD_EXTEND_MIXIN } from "@/mixins/control-field-extend-mixin";
import { DROPDOWN_DATA_MODES } from "@/configs/control-config-enum";
import ListItem from "@/libraries/list-item.class";

/**
 * Dropdown Control.
 * I've been thinking all day, all night, should I use some library (select2, choices.js,...)
 * But, after some researched via https://bundlephobia.com/ , I decided to use Native Select instead
 * In order to save some KBs, the bundle is kinda bigger now @@
 * @property {ListItem[]} listOptions
 */
export default {
  name: "DropdownControl",
  mixins: [CONTROL_FIELD_EXTEND_MIXIN],
  data: () => ({
    listOptions: [],
    dataMode: "",
    apiURL: "",
    selectedItems: [],
  }),
  watch: {
    control: {
      deep: true,
      handler(controlObj) {
        // we cached it. If nothing change => no reload
        if (
          // single check
          controlObj.dataMode !== this.dataMode ||
          controlObj.apiURL !== this.apiURL ||
          // list check
          (controlObj.dataMode === DROPDOWN_DATA_MODES.list.val &&
            controlObj.items.length !== this.listOptions.length)
        ) {
          return this.retrieveOptionLists();
        }
      },
    },
    value: function () {
        this.selectedItems = this.value;
    }
  },
  methods: {
    optionClicked(val) {
      let selectedValues = this.selectedItems;
      if (selectedValues.map((item) => `${item}`).indexOf(`${val}`) > -1) {
        selectedValues = selectedValues.filter(
          (item) => `${item}` != `${val}`
        );
      } else {
        selectedValues.push(`${val}`);
      }
      this.selectedItems = selectedValues;
      this.updateValue(selectedValues);
    },
    onChange(event, isMultiple) {
      const selectedValue = isMultiple
        ? [...event.target.options]
            .filter((option) => option.selected)
            .map((option) => option.value)
        : event.target.value;
      this.updateValue(selectedValue);
      this.$forceUpdate();
    },
    retrieveOptionLists() {
      this.dataMode = this.control.dataMode;
      this.apiURL = this.control.apiURL;

      // pick up option data
      if (this.control.dataMode === DROPDOWN_DATA_MODES.list.val) {
        this.listOptions = this.control.items;
        return;
      }

      // our code goes down here? => REST-API
      if (!this.control.apiURL) {
        throw new TypeError(
          "[Dropdown] In order to use REST-API for Dropdown. You must set your Rest-API Endpoint."
        );
      }

      let endAPI = this.control.apiURL;
      if (this.control.apiURL.indexOf("{domain}") > -1) {
        endAPI = this.control.apiURL.replace("{domain}", this.baseURL);
      }

      endAPI = endAPI
        .replace(
          /http:\/\/\{getProvidersAPI\}/g,
          `${this.baseURL}providers-basic-info`
        )
        .replace(
          /http:\/\/\{getManagersAPI\}/g,
          `${this.baseURL}/managers-basic-info`
        );

      this.promiseStarted("dropdown-api");

      // ok retrieve now
      fetch(endAPI, {
        method: "GET",
      })
        .then((result) => result.json())
        .then(this.afterRestAPICallDataSuccessfully)
        .catch(this.restAPICallErrorHandling);
    },

    /**
     * [FOR-RestAPI-Dropdown][EVENT]
     * Appending data after success
     * @param {[]} result
     */
    afterRestAPICallDataSuccessfully(result) {
      this.promiseEnded("dropdown-api");
      if (!Array.isArray(result) && !Array.isArray(result.data)) {
        throw new TypeError(
          `[DROPDOWN-${this.control.name}] Wrong API-data format.`
        );
      }

      // clear list
      this.listOptions = [];

      // traversal all list and add it into the list
      (result.data || result).forEach((optionObj) => {
        this.listOptions.push(
          new ListItem(`${optionObj[this.valueKey]}`, optionObj[this.textKey])
        );
      });
    },

    /**
     * [FOR-RestAPI-Dropdown][EVENT]
     * Show error for dropdown.
     */
    restAPICallErrorHandling(e) {
      this.promiseEnded("dropdown-api");
      console.error(
        `[DROPDOWN-Control-${this.control.uniqueId}] Request API to get data failed.`,
        e
      );
    },
  },

  computed: {
    /**
     * [API-Only] Get Text Key
     * @returns {string}
     */
    textKey() {
      return this.control.apiTextKey || "text";
    },

    /**
     * [API-Only] Get Text Key
     * @returns {string}
     */
    valueKey() {
      return this.control.apiValueKey || "value";
    },
  },

  created() {
    this.retrieveOptionLists();
  },
};
</script>

<style scoped>

.dropdown-control-feild .multi-select {
    display: inline-block;
    height: auto;
}
.dropdown-control-feild .multi-select .option {
    margin: 10px;
    border: 1px solid #cacaca;
    border-radius: 5px;
    padding: 8px;
    float: left;
    cursor: pointer;
}
.dropdown-control-feild .multi-select .option.selected {
    background-color: #E7EFFF
}
</style>
