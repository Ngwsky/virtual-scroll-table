<template>
  <v-simple-table fixed-header
    :height="height"
    :dense="dense"
    ref="vstable"
  >
    <template v-slot:default>
      <thead ref="thead">
        <tr>
          <th v-if="showSelect"><v-simple-checkbox :value="isSelectedAll" :indeterminate="indeterminateSelectedAll" @input="selectAll" :ripple="false"></v-simple-checkbox></th>
          <th v-for="(header, index) in headers"
          :key="index" @click="toggleSortOrder(index)" @mouseover="headerMouseOver(index)" @mouseleave="headerMouseLeave(index)"
          >{{ header }}
            <v-icon class="ml-2" :color="-1 < sortIdxs.findIndex(v => v === index) ? 'blue darken-4' : 'blue-grey lighten-2'" :style="-1 < sortIdxs.findIndex(v => v === index) || hoveredHeaderIdx === index ? 'visibility:visible;' : 'visibility:hidden;'">
              <template v-if="-1 < sortIdxs.findIndex(v => v === index)">
                <template v-if="sortOrders[sortIdxs.findIndex(v => v === index)] == -1">{{ svgChevronDown }}</template>
                <template v-else-if="sortOrders[sortIdxs.findIndex(v => v === index)] == 1">{{ svgChevronUp }}</template>
              </template>
              <template v-else>{{ svgChevronUp }}</template>
            </v-icon>
            <v-chip color="blue-grey lighten-4" small class="px-2" v-if="-1 < sortIdxs.findIndex(v => v === index)">{{ sortIdxs.findIndex(v => v === index) + 1 }}</v-chip>
            <v-menu left offset-y :close-on-content-click="false">
              <template v-slot:activator="{ on, attrs }">
                <v-icon class="float-right" :color="0 < filterValues[index].length ? 'blue darken-4' : 'blue-grey lighten-2'" dense v-bind="attrs" v-on="on">{{ svgFilterVariant }}</v-icon>
              </template>
              <v-card outlined>
                <v-autocomplete clearable deletable-chips multiple small-chips v-model="filterValues[index]" :items="filterSelectOptions[index]" dense class="mx-1"></v-autocomplete>
              </v-card>
            </v-menu>
          </th>
        </tr>
      </thead>
      <tbody>
        <tr v-if="0 < paddingtop">
          <td
            :colspan="showSelect ? headers.length + 1 : headers.length"
            :style="'padding-top:' + paddingtop + 'px'"
          >
          </td>
        </tr>
        <tr v-for="(vitem, viidx) in vitems"
          :key="viidx"
        >
          <td v-if="showSelect"><v-simple-checkbox :value="vitems[viidx][headers.length + 1]" @input="selectRow(viidx)" :ripple="false"></v-simple-checkbox></td>
          <td v-for="(value, hidx) in headers"
            :key="hidx">{{ vitem[hidx] }}</td>
        </tr>
        <tr v-if="0 < paddingbottom">
          <td
            :colspan="showSelect ? headers.length + 1 : headers.length"
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
      default: 0,
    },
    showSelect: {
      type: Boolean,
      required: false,
      default: false,
    },
    multiSort: {
      type: Boolean,
      required: false,
      default: false,
    },
    locale: String,
    dense: {
      type: Boolean,
      required: false,
      default: false,
    },
    value: Array,
  },
  data () {
    return {
      start: 0,
      timeout: null,
      headerHeight: 48,
      rowHeight: 48,
      scrollHeight: 0,
      svgChevronDown: mdiChevronDown,
      svgChevronUp: mdiChevronUp,
      svgFilterVariant: mdiFilterVariant,
      svgSortVariant : mdiSortVariant,
      hoveredHeaderIdx: null,
      orgItems: [],
      filteredItems: [],
      filterSelectOptions: [],
      filterValues: [],
      sortIdxs: [],
      sortOrders: [],
      itemsSelected: [],
      isSelectedAll: false,
      indeterminateSelectedAll: false,
      collator: null,
    }
  },
  created() {
    this.itemsSelected = this.items.map(() => false);
    this.orgItems = this.items.map((item, index) => {
      let r = item.slice();
      r.push(index); // orgItemIdx
      r.push(false); // isSelected
      return r;
    });
    Object.freeze(this.orgItems);
    this.filteredItems = this.orgItems.slice();
    Object.freeze(this.filteredItems);
    this.filterSelectOptions = this.headers.map((header, index) => Array.from(new Set(this.orgItems.map(item => item[index]))));
    this.filterValues = this.headers.map(() => []);
  },
  mounted() {
    this.$refs.vstable.$el.childNodes[0].addEventListener("scroll", this.onScroll);
    this.headerHeight = this.$refs.thead.getBoundingClientRect().height;
    this.rowHeight = this.$refs.thead.getBoundingClientRect().height; // ホントならtbodyの一行から取得？でも一行の高さを固定にしないとおかしなことになるから、とりあえずはヘッダーの高さで。
    this.scrollHeight = this.$refs.vstable.$el.childNodes[0].scrollHeight;
  },
  methods: {
    onScroll(e) {
      this.timeout && clearTimeout(this.timeout);
      this.timeout = setTimeout(() => {
        const { scrollTop } = e.target;
        let rows = Math.ceil((scrollTop - this.rowHeight) / this.rowHeight) - Math.floor(this.bench / 2);
        if (rows < 0) rows = 0;
        this.start = rows + this.rowsPerPage > this.filteredItems.length ?
          this.filteredItems.length - this.rowsPerPage: rows;
        this.$nextTick(() => {
          e.target.scrollTop = scrollTop;
          this.scrollHeight = this.$refs.vstable.$el.childNodes[0].scrollHeight;
        });
      }, 20);
    },
    headerMouseOver(index){
      this.hoveredHeaderIdx = index;
    },
    headerMouseLeave(){
      this.hoveredHeaderIdx = null;    
    },
    toggleSortOrder(idx){
      if (this.multiSort) {
        let targetIdx = this.sortIdxs.findIndex(v => v === idx);
        if (targetIdx === -1) {
          this.sortIdxs.push(idx);
          this.sortOrders.push(1);
        } else {
          if (this.sortOrders[targetIdx] === -1) {
            this.sortIdxs.splice(targetIdx, 1);
            this.sortOrders.splice(targetIdx, 1);
            if (this.sortIdxs.length === 0) {
              this.filter(this.filterValues);
              return;
            }
          } else if (this.sortOrders[targetIdx] === 1) {
            this.sortOrders[targetIdx] = -1;
          }
        }
      } else {
        if (this.sortIdxs && 0 < this.sortIdxs.length && this.sortIdxs[0] === idx) {
          if (this.sortOrders[0] === -1) {
            this.sortIdxs = [];
            this.sortOrders = [];
            this.filter(this.filterValues);
            return;
          } else if (this.sortOrders[0] === 1) {
            this.sortOrders = [-1];
          } else {
            this.sortOrders = [1];
          }
        } else {
          this.sortIdxs = [idx];
          this.sortOrders = [1];
        }
      }
      this.sort();
    },
    filter(vals){
      this.filteredItems = this.orgItems.filter(item => {
        for (let i = 0; i < vals.length; i++) {
          if (vals[i] && 0 < vals[i].length) {
            if (vals[i].findIndex(v => v === item[i]) === -1) return false;
          }
        }
        return true;
      });
      this.refreshSelectAll(this.filteredItems);
      this.refreshFilterSelections(this.filteredItems);
      this.sort(this.sortOrders);
    },
    refreshFilterSelections(val){
      this.filterSelectOptions = this.headers.map((header, index) => {
        if (this.filterValues[index] && 0 < this.filterValues[index].length) return this.filterSelectOptions[index];
        return Array.from(new Set(val.map(item => item[index])));
      });
    },
    refreshSelectAll(val){
      if (val) {
        let firstVal = val[0][this.headers.length + 1];
        if (val.findIndex(v => v[this.headers.length + 1] !== firstVal) === -1) {
          this.isSelectedAll = firstVal;
          this.indeterminateSelectedAll = false;
        } else {
          this.isSelectedAll = false;
          this.indeterminateSelectedAll = true;
        }
      }
    },
    sort() {
      if (!this.sortIdxs || this.sortIdxs.length < 1 || this.sortOrders[0] === 0) return;
      if (!this.collator) this.collator = this.locale ? new Intl.Collator(this.locale) : new Intl.Collator('ja');
      this.filteredItems = this.filteredItems.sort((a, b) => {
        for (let i = 0; i < this.sortIdxs.length; i++) {
          if (typeof a[this.sortIdxs[i]] == 'string' && typeof b[this.sortIdxs[i]] == 'string') {
            let c = this.collator.compare(a[this.sortIdxs[i]], b[this.sortIdxs[i]]);
            if (c !== 0) {
              return c * this.sortOrders[i];
            }
          } else {
            if (a[this.sortIdxs[i]] != b[this.sortIdxs[i]]) return (a[this.sortIdxs[i]] - b[this.sortIdxs[i]]) * this.sortOrders[i];
          }
        }
        return 0;
      });
    },
    selectAll(){
      let newVal = this.indeterminateSelectedAll ? true : !this.isSelectedAll;
      this.isSelectedAll = newVal;
      this.indeterminateSelectedAll = false;
      this.filteredItems = this.filteredItems.map(fi => {
        this.itemsSelected[fi[this.headers.length]] = newVal;
        fi[this.headers.length + 1] = newVal;
        return fi;
      });
      this.$emit('input', this.items.filter((item, index) => this.itemsSelected[index]));
    },
    selectRow(vindex){
      let newVal = !this.itemsSelected[this.vitems[vindex][this.headers.length]];

      this.itemsSelected[this.vitems[vindex][this.headers.length]] = newVal;
      this.filteredItems[this.start + vindex][this.headers.length + 1] = newVal;
      this.filteredItems.splice();

      if (this.filteredItems.findIndex(v => v !== newVal) === -1) {
        this.isSelectedAll = newVal;
        this.indeterminateSelectedAll = false;
      } else {
        this.isSelectedAll = false;
        this.indeterminateSelectedAll = true;
      }

      this.$emit('input', this.items.filter((item, index) => this.itemsSelected[index]));
    },
  },
  watch: {
    items: function(val) {
      this.orgItems = this.items.map((item, index) => {
        let r = item.slice();
        r.push(index); // orgItemIdx
        r.push(false); // isSelected
        return r;
      });
      Object.freeze(this.orgItems);
      this.filteredItems = this.orgItems.slice();
      Object.freeze(this.filteredItems);
      this.filterSelectOptions = this.headers.map((header, index) => Array.from(new Set(val.map(item => item[index]))));
      this.filterValues = this.headers.map(() => []);
      this.$nextTick(() => {
        this.scrollHeight = this.$refs.vstable.$el.childNodes[0].scrollHeight;
      });
    },
    locale: function(val) {
      this.collator = val ? new Intl.Collator(val) : new Intl.Collator();
    },
    filterValues: function(val) {
      this.filter(val);
    },
  },
  computed: {
    rowsPerPage() {
      return Math.floor(this.height / this.rowHeight) - 1;
    },
    vitems() {
/*
      let s = Date.now();
      console.log('vitems start');
*/
      let r = this.filteredItems.slice(this.start, this.start + this.rowsPerPage + this.bench);
      Object.freeze(r);
/*
      console.log('vitems end', (Date.now() - s) / 1000.0, "sec");
*/
      return r;
    },
    paddingtop() {
      return this.start * this.rowHeight;
    },
    paddingbottom() {
      return this.rowHeight * (this.filteredItems.length - (this.start + this.rowsPerPage + this.bench));
    },
  },
}
</script>
