<template>
   <v-container>
   <v-slider
     v-model="slider"
     class="align-center"
     :max="100000"
     :min="0"
     hide-details
   >
      <template v-slot:append>
        <v-text-field
         v-model="slider"
         class="mt-0 pt-0"
         hide-details
         single-line
         type="number"
         style="width: 60px"
         ></v-text-field>
      </template>
   </v-slider>
   <VirtualScrollTable v-model="selected" :height="480" :bench="18" :headers=headers :items=items show-select multi-sort />
   </v-container>
</template>

<script>
import VirtualScrollTable from '../components/VirtualScrollTable'

export default {
   name: 'Home',
   components: {
      VirtualScrollTable
   },
   data () {
    return {
       slider: 9999,
       headers: ['ID', 'Name', 'Type', 'Value', '日本語'],
       items: [],
       types: ['Bank transfer', 'Cash', 'Credit card', 'Coupon', 'Family mart'],
       selected: [],
    }
  },
  methods: {
     refreshItems: function(val){
        let temp = [];
        let b = Date.now();
        let kana = "あいうえおかきくけこさしすせそたちつてとなにぬねのはひふへほまみむめもやゆよらりるれろわゐうゑをんアイウエオカキクケコサシスセソタチツテトナニヌネノハヒフヘホマミムメモヤユヨラリルレロワヰウヱヲンがぎぐげござじずぜぞだぢづでどばびぶべぼヴガギグゲゴザジズゼゾダヂヅデドバビブベボぱぴぷぺぽパピプペポぁぃぅぇぉっゃゅょゎァィゥェォヵヶッャュョー";
        console.log("start generate data ...");
        for (let i = 1; i <= val; i++) {
           temp.push([i, Math.random().toString(36).slice(-8), this.types[Math.floor(Math.random() * this.types.length)], Math.floor(Math.random() * 999999),
              Array(2 + Math.floor(Math.random() * 5)).fill().map(() => {
                let s = Math.floor(Math.random() * kana.length);
                return kana.substring(s, s + 1);
              }).join('')
              ]);
        }
        console.log("complete", (Date.now() - b) / 1000.0, "sec");
        this.items = temp;
     }
  },
  created(){
    this.refreshItems(this.slider);
  },
  watch: {
    slider: function (val) {
      this.refreshItems(val);
    },
    selected: function(val) {
      console.log("selected", val);
    }
  },
}
</script>
