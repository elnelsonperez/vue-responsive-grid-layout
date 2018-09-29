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
                @layout-init="initLayout"
                @width-change="changeWidth"
                @breakpoint-change="changeBreakpoint"
                :layouts="layout"
                :cols="cols"
                :compact-type="'vertical'"
                :vertical-compact="true"
                :init-on-start="true"
                :breakpoint="breakpoint"
                :breakpoints="breakpoints"
                :cols-all="colsAll"
                ref="grid"
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
        layout: {
          "lg": [
            { x: 0, y: 0, w: 2, h: 9, i: "1"},
            { x: 2, y: 0, w: 2, h: 6, i: "2"},
            { x: 4, y: 0, w: 2, h: 8, i: "3"},
            { x: 0, y: 3, w: 2, h: 5, i: "4"},
          ]
        },
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
        return this.layout;
      }
    },
    methods: {
      initLayout({layout, cols}) {
        this.cols = cols;
      },
      changeWidth({width, newCols}) {
        this.containerWidth = width;
        this.cols = newCols;
      },
      updateLayout({layout, breakpoint}) {
        let filtered;
        filtered = layout.map( (item) => { return { x: item.x, y: item.y, w: item.w, h: item.h, i: item.i }})
        this.layout[breakpoint] = filtered;
      },
      changeBreakpoint({breakpoint, cols}) {
        this.cols = cols;
        this.breakpoint = breakpoint;
      },
      changeLayout({layout, breakpoint}) {
        let filtered;
        filtered = layout.map( (item) => { return { x: item.x, y: item.y, w: item.w, h: item.h, i: item.i }})
        this.layout[breakpoint] = filtered;
      },
      gridMode() {
        this.$refs.grid.resizeAllItems(false, false);
      },
      listMode() {
        this.$refs.grid.resizeAllItems(true, false);
      },
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
