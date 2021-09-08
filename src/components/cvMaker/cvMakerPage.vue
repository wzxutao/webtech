<template>
  <div class="A4paper" :ref="thisPageRef" :id="thisPageId" @keyup="checkHeightOverflow">
  <mainPage v-if="pageType === 'main'" />
  <subPage v-if="pageType === 'sub'" />
  </div>
</template>

<script>
import mainPage from "@/components/cvMaker/cvMakerMainPage";
import subPage from "@/components/cvMaker/cvMakerSubPage";

export default {
  props:['pageId', 'pageType'],
  computed:{
    thisPageRef(){
      return `cvPage${this.thisPageId}`;
    },
    thisPageId: () => this && this.pageId ? this.pageId : -1,
  },
  components:{
    mainPage,
    subPage,
  },
  methods:{
    // do something if content has overflown the height of this page
    checkHeightOverflow(){
      // const elem = this.$refs[this.thisPageRef];
      // if(elem.scrollHeight > elem.clientHeight){
      //   this.$alert(`Page ${this.thisPageId} is full. Please delete some contents`,'Error','error');
      // }
    }
  },

}

</script>

<style>
  .cv-contents i{
    font: inherit;
    font-style: italic;
  }

  .cv-contents b{
    font: inherit;
    font-weight: bold;
  }

  .cv-contents larger{
    font: inherit;
    font-size: 125%;
  }

  .cv-contents smaller{
    font: inherit;
    font-weight: 80%;
  }

  </style>


