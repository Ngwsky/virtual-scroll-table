<template>
  <v-simple-table fixed-header
    :height="height"
    ref="vstable"
  >
    <template v-slot:default>
      <thead ref="thead">
        <tr>
          <th v-if="showSelect"><v-simple-checkbox :value="isSelectedAll" :indeterminate="indeterminateSelectedAll" @input="selectAll"></v-simple-checkbox></th>
          <th v-for="(header, index) in headers"
          :key="index" @click="sort(index)"
          >{{ header }}
            <v-menu offset-y :close-on-content-click="false">
              <template v-slot:activator="{ on, attrs }">
                <v-icon :color="fvals[index] ? 'blue darken-4' : 'blue-grey'" dense v-bind="attrs" v-on="on">{{ svgFilterVariant }}</v-icon>
              </template>
              <v-card outlined>
                <v-autocomplete clearable v-model="fvals[index]" :items="filters[index]" dense></v-autocomplete>
              </v-card>
            </v-menu>
            <v-icon v-if="sortidx == index"><template v-if="sortorder == -1">{{ svgChevronDown }}</template><template v-else-if="sortorder == 1">{{ svgChevronUp }}</template></v-icon>
          </th>
        </tr>
      </thead>
      <tbody>
        <tr v-if="start > 0">
          <td
            :colspan="headers.length"
            :style="'padding-top:' + paddingtop + 'px'"
          >
          </td>
        </tr>
        <tr v-for="(vitem, viidx) in vitems"
          :key="viidx"
        >
          <td v-if="showSelect"><v-simple-checkbox :value="vitems[viidx].isSelected" @input="selectRow(viidx)"></v-simple-checkbox></td>
          <td v-for="(value, hidx) in headers"
            :key="hidx">{{ vitem[hidx] }}</td>
        </tr>
        <tr v-if="start + rowsPerPage < filteredItems.length">
          <td
            :colspan="headers.length"
            :style="'padding-bottom:' + paddingbottom + 'px'"
          >
          </td>
        </tr>
      </tbody>
    </template>
  </v-simple-table>
</template>

<script>
import { mdiFilterVariant, mdiChevronDown, mdiChevronUp, mdiSortVariant } from '@mdi/js';

export default {
  name: 'VirtualScrollTable',
  props: {
    height: Number,
    headers: Array,
    items: Array,
    bench: {
      type: Number,
      default: 0
    },
    showSelect: Boolean,
    value: Array,
  },
  data () {
    return {
      start: 0,
      timeout: null,
      headerHeight: 48,
      rowHeight: 48,
      svgChevronDown: mdiChevronDown,
      svgChevronUp: mdiChevronUp,
      svgFilterVariant: mdiFilterVariant,
      svgSortVariant : mdiSortVariant,
      filteredItems: [],
      filters: [],
      fvals: [],
      sortidx: null,
      sortorder: 0,
      itemsSelected: [],
      isSelectedAll: false,
      indeterminateSelectedAll: false,
    }
  },
  mounted() {
    this.$refs.vstable.$el.childNodes[0].addEventListener("scroll", this.onScroll);
    this.headerHeight = this.$refs.thead.getBoundingClientRect().height;
    this.rowHeight = this.$refs.thead.getBoundingClientRect().height; // ホントならtbodyの一行から取得？でも一行の高さを固定にしないとおかしなことになるから、とりあえずはヘッダーの高さで。
    this.itemsSelected = this.items.map(() => false);
    this.filteredItems = this.items.map((item, index) => { 
      item.orgItemIdx = index;
      item.isSelected = false;
      return item;
    });
    this.filters = this.headers.map((header, index) => Array.from(new Set(this.items.map(item => item[index]))));
    this.fvals = this.headers.map(() => null);
  },
  methods: {
    onScroll(e) {
      this.timeout && clearTimeout(this.timeout); 
      this.timeout = setTimeout(() => {
        const { scrollTop } = e.target;
        let rows = Math.ceil((scrollTop - this.rowHeight) / this.rowHeight);
        if (rows < 0) rows = 0;
        this.start = rows + this.rowsPerPage > this.filteredItems.length ?
          this.filteredItems.length - this.rowsPerPage: rows;
        this.$nextTick(() => {
          e.target.scrollTop = scrollTop;
        });
      }, 20);
    },
    sort(idx){
      if (this.sortidx == null || this.sortidx !== idx) {
        this.sortidx = idx;
        this.sortorder = 1;
      } else {
        if (this.sortorder === -1) {
          this.sortidx = null;
          this.sortorder = 0;
        } else if (this.sortorder === 1) {
          this.sortorder = -1;
        } else {
          this.sortorder = 1;
        }
      }
    },
    filter(vals){
      this.filteredItems = this.items.filter(item => {
        for (let i = 0; i < vals.length; i++) {
          if (vals[i] && vals[i] !== item[i]) {
            return false;
          }
        }
        return true;
      });
    },
    selectAll(){
      let newVal = this.indeterminateSelectedAll ? true : !this.isSelectedAll;
      this.isSelectedAll = newVal;
      this.indeterminateSelectedAll = false;
      this.filteredItems = this.filteredItems.map(fi => {
        this.itemsSelected[fi.orgItemIdx] = newVal;
        fi.isSelected = newVal;
        return fi;
      });
      this.$emit('input', this.items.filter((item, index) => this.itemsSelected[index]));
    },
    selectRow(vindex){
      let newVal = !this.itemsSelected[this.vitems[vindex].orgItemIdx];

      this.itemsSelected[this.vitems[vindex].orgItemIdx] = newVal;
      this.filteredItems[this.start + vindex].isSelected = newVal;
      this.filteredItems.splice();

      if (this.filteredItems.findIndex(v => v !== newVal) === -1) {
        this.isSelectedAll = newVal;
        this.indeterminateSelectedAll = false;
      } else {
        this.isSelectedAll = false;
        this.indeterminateSelectedAll = true;
      }

      this.$emit('input', this.items.filter((item, index) => this.itemsSelected[index]));
    }
  },
  watch: {
    items: function(val) {
      this.filteredItems = this.items.map((item, index) => { 
        item.orgItemIdx = index;
        item.isSelected = false;
        return item;
      });
      this.filters = this.headers.map((header, index) => Array.from(new Set(val.map(item => item[index]))));
      this.fvals = this.headers.map(() => null);
    },
    sortorder: function(val) {
      if (val === 0) {
        this.filter(this.fvals);
      } else {
        this.filteredItems = this.filteredItems.sort((a, b) => a[this.sortidx] < b[this.sortidx] ? val * -1: val);
      }
    },
    fvals: function(val) {
      this.filter(val);
    },
    filteredItems: function(val) {
      this.filters = this.headers.map((header, index) => Array.from(new Set(val.map(item => item[index]))));
      if (val) {
        let firstVal = val[0].isSelected;
        if (val.findIndex(v => v.isSelected !== firstVal) === -1) {
          this.isSelectedAll = firstVal;
          this.indeterminateSelectedAll = false;
        } else {
          this.isSelectedAll = false;
          this.indeterminateSelectedAll = true;
        }
      }
    }
  },
  computed: {
    rowsPerPage() {
      return Math.floor((this.height - this.rowHeight) / this.rowHeight) + this.bench;
    },
    vitems() {
      return this.filteredItems.slice(this.start, this.rowsPerPage + this.start);
    },
    paddingtop() {
      return this.start * this.rowHeight;
    },
    paddingbottom() {
      return this.rowHeight * (this.filteredItems.length - this.start - this.rowsPerPage);
    },
  },
}
</script>
