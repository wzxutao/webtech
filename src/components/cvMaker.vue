<template>
<div class="cv-back">
  <div class="sidenav">
    <download-button :inputTime='this.downloadExpectedTime' />
    <div>
      <div @click="saveProgress" class="sameLineDisplay">
        <save-button/>
      </div>
      <div @click="loadSavedData" class="sameLineDisplay">
        <load-button/>
      </div>
    </div>
    <div>
      <div @click="addSubPage" class="sameLineDisplay">
        <add-button/>
      </div>
      <div @click="deleteLastSubPage" class="sameLineDisplay">
        <remove-button/>
      </div>
    </div>
    <div>
      <div @click="insertElement" class="sameLineDisplay">
        <add-element-button/>
      </div>
      <div @click="deleteElement" class="sameLineDisplay">
        <remove-element-button/>
      </div>
    </div>
  <div>
    <div  @click="increaseFontSize"   class="sameLineDisplay">
      <big-font-button ref="increase"/>
    </div>
    <div @click="decreaseFontSize" class="sameLineDisplay">
      <small-font-button ref="decrease"/>
    </div>
  </div>
    <div>
      <div @click="boldifyText" class="sameLineDisplay">
        <bold-font-button ref="bold"/>
      </div>
      <div @click="italicizeText" class="sameLineDisplay">
        <incline-font-button ref="incline"/>
      </div>
    </div>
    <button class="btn btn-secondary" @click="previewPrint">
      Preview(dev)
    </button>
</div>

  <!-- cv contents -->
  <div class="cv" ref="cv" @mousemove="handleMousemove" @click="handleMouseClick" @mouseup="handleMouseup">
    <div class="cv-contents" ref="cv-contents">
      <cvPage pageId=0 pageType="main" />

    </div>
  </div>
</div>
</template>


<script>
import downloadButton from "@/components/button/downloadButton";
import addButton from "@/components/button/addPageButton";
import removeButton from "@/components/button/removePageButton";
import addElementButton from "@/components/button/addElementButton";
import removeElementButton from "@/components/button/removeElementButton";
import bigFontButton from "@/components/button/bigFontButton";
import smallFontButton from "@/components/button/smallFontButton";
import boldFontButton from "@/components/button/boldFontButton";
import saveButton from "@/components/button/saveButton";
import loadButton from "@/components/button/loadButton";
import inclineFontButton from "@/components/button/inclineFontButton";
import cvPage from "@/components/cvMaker/cvMakerPage";
import Vue from 'vue';

const MODE_EDIT           = 1,
      MODE_INSERT         = 2,
      MODE_DELETE         = 3,
      MODE_ITALICISE      = 4,
      MODE_BOLDIFY        = 5,
      MODE_INC_FONT_SIZE  = 6,
      MODE_DEC_FONT_SIZE  = 7;

// returns the innermost elem that has the class 'clonable'.
// returns null if 'clonable' is not found before reaching the 'section' tag
function getClonable(elem){
  let elemCurr = elem;
  while(elemCurr !== undefined){
    if(elemCurr.tagName === 'section'){
      return null;
    }

    let classList = elemCurr.classList;
    if(classList === undefined) return null;

    if(classList.contains('clonable')){
      return elemCurr;
    }
    elemCurr = elemCurr.parentNode;
  }
  return null;
}

async function getElemByRef(ref, vue){
  return new Promise((resolve) => {
    let elems = vue.$refs[ref];
    if(elems) resolve(elems);
  })
}

import {
  bus
} from '@/view/index/main';

export default {
  name: 'cv',
  props: {
    // templateId: Number,
    // fetchSavedData: Boolean,
  },
  data() {
    return {
      mode: MODE_EDIT,
      templateId: 0,
      fetchSavedData: false,

      maxPageId: 0,
      // This parameter should call the thread status of the system about the download feedback
      downloadExpectedTime: 5000,

      // temp variables for insertion mode
      elemCurr: null,  // elem the cursor pointing at when the event happened
      elemNew: null,  // pre-inserted new elem
      isCursorAtUpperPart: null,
      // igoreMousemove: false,
      // temp var for deletion
      elemToDelete: null,

    }

  },
  components: {
    downloadButton,
    saveButton,
    loadButton,
    addButton,
    removeButton,
    addElementButton,
    removeElementButton,
    bigFontButton,
    smallFontButton,
    boldFontButton,
    inclineFontButton,
    cvPage,
  },
  methods: {
    animateProgressSaved(){
      this.$confirm("", "Saved", "success");
    },
    async previewPrint(){
      let cvContents = await getElemByRef('cv-contents', this);
      document.write(`
          <html>
          <head>
            <link href="/css/A4Paper.css" rel="stylesheet">
            <link href="/api/template/template.css?id=${this.templateId}" rel="stylesheet">
          </head>
          <body>
            <div class="cv-contents">${cvContents.innerHTML}</div>
          </body>
          </html>`);
    },
    // returns null on succees
    async saveProgress(){
      let cvContents = await getElemByRef('cv-contents', this);
      let avatarImg = document.getElementById('avatar-img');
      avatarImg = avatarImg !== null ? avatarImg.src : ""

      let reqBody = {
        // no longer needs to send the headers, style = templateId.css + A4Paper.css
        htmlHeaders: null,
        cvContents: cvContents.innerHTML,
        templateId: this.templateId,
        avatarUrl: avatarImg,
      }

      try{
        let res = await this.$http.post('/api/cvMaker/save', reqBody);
        if(res.status === 201){
          this.animateProgressSaved();
          return null;
        }
        else{
          this.$alert('save failed',"Error",'error');
        }
      }catch(err){
        if(err.status === 401){
          return this.$alert('Please log in first','Error','error');
        }
        console.log(err);
        this.$alert('save failed','Error','error');
        return err;
      }
    },
    async loadSavedData(){
      try{
        let res = await this.$http.get('/api/cvMaker/load');
        let cvContents = res.body.cvContents;
        // let avatarUrl = res.body.avatarUrl;

        // load template
        this.templateId = res.body.templateId;
        await this.fetchTemplate();

        document.querySelectorAll(".cv-contents")[0].innerHTML = cvContents;
      }catch(err) {
        this.$alert("load failed", "error", "error")
      }
          
    },
    async generatePdf() {
      // for style, you only need ${templateId}.css and A4Paper.css
      let rv = await this.saveProgress();
      if(rv !== null) return; // save failed

      try{
        let res = await this.$http.get('/api/toPdf', {
            responseType: 'arraybuffer'
          });
        let blob = new Blob([res.data]);
        let link = document.createElement('a');
        link.href = window.URL.createObjectURL(blob);
        link.download = "cv.pdf";
        link.click();
      }catch(err){
        console.log(err);
        this.$alert('Failed','Error','error');
      }

    },
    addSubPage(){
      if(this.maxPageId >= 5) return this.$alert('A concise CV is a good CV.','Warning','warning');

      this.$alert("Currently, extra pages won't be printed as HRs rarely read multi-page CVs.", "Warning", "warning")
      const cvPageClass = Vue.extend(cvPage);
      let newPage = new cvPageClass({
        propsData:{
          pageId: ++this.maxPageId,
          pageType: 'sub',
        }
      });
      newPage.$mount();
      this.$refs['cv-contents'].appendChild(newPage.$el);

      newPage.$el.scrollIntoView(true);
    },
    deleteLastSubPage(){
      // does not delete the main page
      if(this.maxPageId === 0) return;
      // removes the last sub page
      let pages = this.$refs['cv-contents'].childNodes;
      let realPageCount = pages.length -1;
      if(realPageCount !== this.maxPageId){
        this.maxPageId = realPageCount;
      }
      let lastPage = pages[this.maxPageId--];
      lastPage.parentNode.removeChild(lastPage);
      lastPage.__vue__.$destroy();
    },
    insertElement(){
      this.mode = MODE_INSERT;
    },
    deleteElement(){
      this.mode = MODE_DELETE;
    },
    handleMouseClick(){
      if(this.mode === MODE_EDIT) return;

      if(this.mode === MODE_INSERT){
        if(this.elemNew){
          this.elemNew.classList.remove('preview');
          this.elemNew = null;
          this.elemCurr = null;
          this.isCursorAtUpperPart = null;
        }
      }else if(this.mode === MODE_DELETE){
        if(this.elemToDelete){
          let parentNode = this.elemToDelete.parentNode;
          if(parentNode){
            parentNode.removeChild(this.elemToDelete);
          }
        }
      }else{
        return;
      }

      this.mode = MODE_EDIT;
    },
    handleInsertion(ev){
      let cursorX = ev.clientX;
      let cursorY = ev.clientY;

      let elemAtCursor = document.elementFromPoint(cursorX, cursorY);
      // get the enclosing clonable block
      let clonable = getClonable(elemAtCursor);
      if(clonable === null) return;

      // if its still the same block as the last operation
      if(clonable === this.elemCurr || clonable === this.elemNew) {
        let rect = this.elemCurr.getBoundingClientRect();
        let isCursorAtUpperPart = cursorY < (rect.top + rect.bottom) / 2;
        if(this.isCursorAtUpperPart === isCursorAtUpperPart) return;
        this.isCursorAtUpperPart = isCursorAtUpperPart;
      }

      // if it's a new position
      this.elemCurr = clonable;
      // check cursor relative position to the clonable block (upper/bottom part)
      let rect = clonable.getBoundingClientRect();
      let isCursorAtUpperPart = cursorY < (rect.top + rect.bottom) / 2;

      // remove the pre-inserted elem when the cursor moves to somewhere else
      if(this.elemNew){
        let parentNode = this.elemNew.parentNode;
        if(parentNode){
          parentNode.removeChild(this.elemNew);
        }
      }

      // create insert a new block the same as the current one for preview
      let newElem = clonable.cloneNode(true); // clones the node and all its descendants
      newElem.classList.add('preview');
      if(isCursorAtUpperPart){
        clonable.insertAdjacentElement('beforebegin', newElem);
      }else{
        clonable.insertAdjacentElement('afterend', newElem);
      }
      this.elemNew = newElem;
      // // ignore mousemove event for some time
      // this.igoreMousemove = true;
      // window.setTimeout(() => this.igoreMousemove = false, 100);

    },
    handleDeletion(ev){
      let cursorX = ev.clientX;
      let cursorY = ev.clientY;

      let elemAtCursor = document.elementFromPoint(cursorX, cursorY);
      // get the enclosing clonable block
      let clonable = getClonable(elemAtCursor);
      if(clonable === null) return;
      if(clonable === this.elemToDelete) return;

      if(this.elemToDelete){
        this.elemToDelete.classList.remove('to-be-deleted');
      }
      clonable.classList.add('to-be-deleted');
      this.elemToDelete = clonable;
    },
    handleMousemove(ev){
      // if(this.igoreMousemove) return;
      if(this.mode === MODE_EDIT) return;

      if(this.mode === MODE_INSERT) return this.handleInsertion(ev);
      if(this.mode === MODE_DELETE) return this.handleDeletion(ev);
    },
    // template html
    async fetchTemplateContent(){
      if(typeof this.templateId === "undefined") return;
      try{
        let res = await this.$http.get(`/api/template/templateContent.html?id=${this.templateId}`);
        document.getElementById("cv-main-page").innerHTML = res.body;
        console.log("template content loaded")
      }catch(err){
        if(err.status === 404) return;
        this.$alert("template content loading failed", "Error", "Error")
      }

    },
    fetchTemplate(){
      const templateElemId = 'cv-template'
      if(typeof this.templateId === "undefined") return;
      // removing existing template
      let existingTemplates = document.querySelectorAll(`#${templateElemId}`);
      for(let templateNode of existingTemplates){
        document.head.removeChild(templateNode);
      }
      // add template
      const styleElemHTML = `<link id="${templateElemId}" rel="stylesheet" href="${this.templatePath}">`
      document.head.insertAdjacentHTML('beforeend', styleElemHTML);
      // perhaps find a better way
      // this.$forceUpdate();
      // console.log('template applied.');
    },
    recoverAllButtons(){
      this.$refs.incline.recovery();
      this.$refs.bold.recovery();
      this.$refs.increase.recovery();
      this.$refs.decrease.recovery();
    }
    ,
    italicizeText(){
      this.recoverAllButtons();
      if(this.mode !== MODE_ITALICISE){
        this.mode = MODE_ITALICISE;
        this.$refs.incline.active();
      }else{
        this.mode = MODE_EDIT;
      }

    },
    boldifyText(){
      this.recoverAllButtons();
      if(this.mode !== MODE_BOLDIFY){
        this.mode = MODE_BOLDIFY;
        this.$refs.bold.active();
      }else{
        this.mode = MODE_EDIT;
      }
    },
    increaseFontSize(){
      this.recoverAllButtons();
      if(this.mode !== MODE_INC_FONT_SIZE){
        this.mode = MODE_INC_FONT_SIZE;
        this.$refs.increase.active();
      }else{
        this.mode = MODE_EDIT;
      }
    },
    decreaseFontSize(){
      this.recoverAllButtons();
      if(this.mode !== MODE_DEC_FONT_SIZE){
        this.mode = MODE_DEC_FONT_SIZE;
        this.$refs.decrease.active();
      }else{
        this.mode = MODE_EDIT;
      }
    },
    handleMouseup(){
      // console.log('mouseup');
      // console.log(this.mode);
      let tag = null;
      switch(this.mode){
        case MODE_ITALICISE:
          tag = 'i'; break;
        case MODE_BOLDIFY:
          tag = 'b'; break;
        case MODE_INC_FONT_SIZE:
          tag = 'larger'; break;
        case MODE_DEC_FONT_SIZE:
          tag = 'smaller'; break;

        default: return;
      }

      const selection = window.getSelection();
      // console.log(selection);
      const selectedText = selection.toString();

      const range = selection.getRangeAt(0);
      range.deleteContents();
      let elem = document.createElement(tag);
      elem.textContent = selectedText;
      range.insertNode(elem);
    }
  },
  computed: {
    templatePath() {
      const url = '/api/template/template.css';
      const query = `?id=${this.templateId}`;
      return url + query;
    }
  },
  async created() {
    // fetch saved data
    const query = this.$router.currentRoute.query;
    this.templateId = query.templateId ? query.templateId : 0;
    this.fetchSavedData = query.fetchSavedData ? query.fetchSavedData : false;

    if(this.fetchSavedData){
      this.loadSavedData();
    }else{
      await this.fetchTemplateContent();
      await this.fetchTemplate();
    }

    bus.$on('downloadAsPdfClick', this.generatePdf);
    bus.$on('forceUpdate', this.$forceUpdate);
  },
}
</script>

<style src='../view/index/assets/cvMaker.css'>
</style>
<style>
  @media print {
    .sidenav {
      display: none;
    }


  }
</style>
