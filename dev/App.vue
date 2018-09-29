<template>
    <div id="wrapper" style="align-items: stretch;width: 100%; height: 100%;">

        <div style="height: 30px;">
            <div class="pull-right">
                <div @click="switchLayout()" class="btn btn-md"><span class="glyphicon glyphicon-share-alt"></span></div>
                <div @click="gridMode()" class="btn btn-md"><span class="glyphicon glyphicon-th-large"></span></div>
                <div @click="listMode()" class="btn btn-md"><span class="glyphicon glyphicon-list"></span></div>
            </div>
        </div>
        <vue-responsive-grid-layout
                @layout-update="updateLayout"
                @layout-change="changeLayout"
                @layout-switched="onLayoutSwitched"
                @layout-ready="readyLayout"
                @layout-init="initLayout"
                @layout-resized="resizedLayout"
                @width-init="initWidth"
                @width-change="changeWidth"
                @breakpoint-change="changeBreakpoint"
                :layouts="currentLayouts"
                :cols="cols"
                :compact-type="'vertical'"
                :vertical-compact="true"
                :init-on-start="false"
                :breakpoint="breakpoint"
                :breakpoints="breakpoints"
                :cols-all="colsAll"
                ref="layout"
        >
            <template slot-scope="props">
                <vue-grid-item
                        v-for="item in props.layout"
                        v-if="item.i && props"
                        :key="item.i"
                        :x="item.x"
                        :y="item.y"
                        :w="item.w"
                        :h="item.h"
                        :i="item.i"
                        :cols="props.cols"
                        :container-width="props.containerWidth"
                        :default-size="2"
                        :is-draggable="isDraggable"
                        :is-resizable="isResizable"
                        :use-css-transforms="true"
                        :height-from-children="false"
                        :can-be-resized-with-all="true"
                >
                    <div class="panel panel-default panel-100-height">
                        <div class="panel-heading text-center background-red">Simple div in slot of grid-item</div>
                        <div class="panel-body"></div>
                    </div>
                </vue-grid-item>
            </template>
        </vue-responsive-grid-layout>
    </div>
</template>

<script type="text/javascript">
  import {VueResponsiveGridLayout, VueGridItem } from '../src/index.js'

  export default {
    components: {
      'vue-responsive-grid-layout': VueResponsiveGridLayout,
      'vue-grid-item': VueGridItem
    },
    data (){

      return {
        layouts: {
          1 : {
            "lg": [
              { x: 0, y: 0, w: 2, h: 9, i: "1"},
              { x: 2, y: 0, w: 2, h: 6, i: "2"},
              { x: 4, y: 0, w: 2, h: 8, i: "3"},
              { x: 0, y: 3, w: 2, h: 5, i: "4"}
            ]
          },
          2: {
            "lg": [
              { x: 0, y: 0, w: 2, h: 7, i: "1"},
              { x: 2, y: 0, w: 2, h: 7, i: "2"},
            ]
          }
        },
        currentLayoutsId: 1,
        breakpoint: "lg",
        cols: 10,
        breakpoints: { lg: 1200, md: 996, sm: 768, xs: 480, xxs: 0 },
        colsAll: { lg: 10, md: 8, sm: 6, xs: 4, xxs: 2 },
        isDraggable: true,
        isResizable: true,
      }
    },
    computed: {
      currentLayouts() {
        return this.layouts[this.currentLayoutsId];
      }
    },
    methods: {
      readyLayout() {
        console.log('layout ready');
        this.$refs.layout.initLayout();
      },
      initLayout({layout, cols}) {
        this.cols = cols;
      },
      initWidth({width}) {
        this.containerWidth = width;
      },
      switchLayout() {
        switch(this.currentLayoutsId) {
          case 1:
            this.currentLayoutsId = 2;
            this.$refs.layout.switchLayout(this.currentLayouts);
            break;
          case 2:
            this.currentLayoutsId = 1;
            this.$refs.layout.switchLayout(this.currentLayouts);
            break;
        }
      },
      onLayoutSwitched() {
        console.log('layouts switched')
      },
      changeWidth({width, newCols}) {
        this.containerWidth = width;
        this.cols = newCols;
      },
      updateLayout({layout, breakpoint}) {
        let filtered;
        filtered = layout.map( (item) => { return { x: item.x, y: item.y, w: item.w, h: item.h, i: item.i }})
        this.layouts[breakpoint] = filtered;
      },
      changeBreakpoint({breakpoint, cols}) {
        this.cols = cols;
        this.breakpoint = breakpoint;
      },
      changeLayout({layout, breakpoint}) {
        let filtered;
        filtered = layout.map( (item) => { return { x: item.x, y: item.y, w: item.w, h: item.h, i: item.i }})
        this.layouts[breakpoint] = filtered;
      },
      gridMode() {
        this.$refs.layout.resizeAllItems(false, false);
      },
      listMode() {
        this.$refs.layout.resizeAllItems(true, false);
      },
      resizedLayout() {
        console.log('layout resized')
      }
    },
  }
</script>

<style>
    html {
        height: 100%;
    }

    body {
        height: 100%;
    }

    #content {
        padding: 0px 20px;
        min-height: 100vh;
        transition: all 0.3s;
        width: 100%;
    }

    .resizable-handle {
        position: absolute;
        width: 20px;
        height: 20px;
        bottom: 0;
        right: 0px;
        text-align: right;
    }

    .resizable-handle::after {
        content: "";
        position: absolute;
        right: 3px;
        bottom: 3px;
        width: 5px;
        height: 5px;
        border-right: 2px solid #FFFFFF;
        border-bottom: 2px solid #FFFFFF;
    }

    .panel-100-height {
        height: 100%;
    }

    .background-red {
        background-color: red !important;
    }
    .vue-responsive-grid-layout {
        display:block;
        position:relative;
    }

</style>
