<template>
    <div :class="classes">
        <slot :containerWidth="containerWidth" :layout="layouts[breakpoint]" :cols="cols">
        </slot>
        <grid-item class="vue-grid-placeholder"
                   :containerWidth="containerWidth"
                   v-show="isDragging || isResizing"
                   :isDraggable="false"
                   :x="placeholder.x"
                   :y="placeholder.y"
                   :w="placeholder.w"
                   :h="placeholder.h"
                   :i="placeholder.i"
                   :placeholder="true"
                   :cols="cols"
                   :rowHeight="rowHeight"
                   style="border:1px dotted #ddd;">
        </grid-item>

        <width-provider
                :selector="providerSelector"
                @widthChange="onWidthChange"
                @widthInit="onWidthInit">
        </width-provider>
    </div>
</template>

<script>
    import WidthProvider from './helpers/WidthProvider.vue'
    import {eventBus} from './event-bus/eventBus.js'
    import GridItem from './VueGridItem.vue'
    import {
        getLayoutItem,
        cloneLayoutItem,
        moveElement,
        compact,
        getAllCollisions,
        cloneLayout,
        synchronizeLayoutWithChildren
    } from './utils/utils'
    import * as _ from "lodash";
    import {
        findOrGenerateResponsiveLayout,
        getBreakpointFromWidth,
        getColsFromBreakpoint
    } from "./utils/ResponsiveUtils";

    export default {
        components: {
            'grid-item' : GridItem,
            'width-provider': WidthProvider
        },
        props: {
            breakpoint: {
                type: String,
                required: false,
                default: 'lg'
            },
            rowHeight: {
                required: false,
                type: Number,
                default: 10
            },
            cols: {
                type: Number,
                required: false,
                default: 12
            },
            layouts: {
                required: true,
                validator: value => {
                    return value instanceof Object || value === null;
                }
            },
            // ("horizontal" | "vertical")
            compactType: {
                type: String,
                required: false,
                default: "vertical"
            },
            preventCollision: {
                type: Boolean,
                required: false,
                default: false
            },
            breakpoints: {
                type: Object,
                required: false,
                default: () => { return { lg: 1200, md: 996, sm: 768, xs: 480, xxs: 0 } }
            },
            colsAll: {
                type: Object,
                required: false,
                default: () => { return { lg: 12, md: 6, sm: 4, xs: 2, xxs: 1 } }
            },
            initOnStart : {
                type: Boolean,
                required: false,
                default: true
            },
            className : {
                required: false,
                type: String,
                default: ""
            },
            providerSelector: {
                required: false,
                type: String
            },
            disabled: {
                required: false,
                type: Boolean,
                default: false
            },
            queueJobs: {
                required: false,
                type: Boolean,
                default: false
            }
        },
        computed: {
            classes() {
                return {
                    "vue-responsive-grid-layout": true,
                    [this.className]: this.className
                }
            },
            ready() {
                return this.inited && this.layouts instanceof Object;
            }
        },
        data() {
            return {
                containerWidth: 0,
                isDragging: false,
                isResizing: false,
                oldDragItem: {},
                oldLayout: [],
                placeholder: {
                    x : 0,
                    y : 0,
                    w : 0,
                    h : 0,
                    i : '-1'
                },
                activeDrag : null,
                inited: false,
                itemsResized: 0,
                jobsQueued: [],
                queueProcessing: false,

            }
        },
        created() {
            eventBus.$on('onDragStart', this.onDragStart);
            eventBus.$on('onDrag', this.onDrag);
            eventBus.$on('onDragStop', this.onDragStop);
            eventBus.$on('onResizeStart', this.onResizeStart);
            eventBus.$on('onResize', this.onResize);
            eventBus.$on('onResizeStop', this.onResizeStop);
            eventBus.$on('updateHeight', this.onHeightUpdate);

            this.$on('layout-ready', this.readyLayout);
            this.$on('width-init', this.widthInited);
        },

        watch: {
            ready(val) {
                if (val) {
                    this.$emit('layout-ready', {width : this.containerWidth});
                }
            },
            jobsQueued(val) {
                if (val) {
                    if (!this.queueProcessing)
                        this.queueProcessSequently()
                }
            }
        },

        methods: {
            queueProcessSequently() {
                this.queueProcessing = true;

                if (this.jobsQueued.length === 0) {
                    this.queueProcessing = false;

                    return;
                }

                let item = this.jobsQueued[0];

                switch (item.job) {
                    case 'updateHeight':
                        this.resizeItem(item.i, item.w, item.h).then(({layout, layouts}) => {
                            this.$emit('layout-item-height-updated', {layouts: _.cloneDeep(layouts), layout: _.cloneDeep(layout), callback: () => {
                                    this.jobsQueued.shift();
                                    this.queueProcessSequently();
                                }});
                        }).catch(err => this.$emit('layout-item-height-updating-failed', JSON.stringify(err)))
                        break;

                    case 'heightUpdated':
                        this.$emit('layout-height-updated', {mode:item.mode});
                        this.jobsQueued.shift();
                        this.queueProcessSequently();
                        break;

                    case 'resizeItem':
                        this.moveItem(item.i, item.x, item.y, item.compactType).then( ({layout}) => {
                            this.resizeItem(item.i, item.w, item.h, layout).then(({layout, layouts}) => {
                                this.$emit('layout-item-resized', {
                                    layouts: _.cloneDeep(layouts), layout: _.cloneDeep(layout), callback: () => {
                                        this.jobsQueued.shift();
                                        this.queueProcessSequently();
                                    }
                                });

                            });

                        }).catch(err => this.$emit('layout-item-resizing-failed', JSON.stringify(err)))
                        break;

                    case 'layoutResized':
                        this.$emit('layout-resized');
                        this.jobsQueued.shift();
                        this.queueProcessSequently();
                        break;
                }


            },
            onWidthInit(width) {
                this.containerWidth = width;
                this.inited = true;
            },
            readyLayout() {
                if (this.initOnStart) {
                    this.initLayout();
                }
            },
            initLayout (mode = true) {
                if (this.inited && this.layouts instanceof Object && this.ready && !this.disabled) {

                    let currentLayouts = _.cloneDeep(this.layouts);
                    let currentCols = this.cols;
                    let currentBreakpoints = _.cloneDeep(this.breakpoints)

                    const breakpoint = getBreakpointFromWidth(currentBreakpoints, this.containerWidth);
                    const cols = getColsFromBreakpoint(breakpoint, this.colsAll);

                    if (this.breakpoint === breakpoint) {
                    } else {
                        this.onWidthChange(this.containerWidth);
                    }

                    let layout = findOrGenerateResponsiveLayout(
                        currentLayouts,
                        currentBreakpoints,
                        breakpoint,
                        breakpoint,
                        cols,
                        this.compactType
                    );

                    layout = synchronizeLayoutWithChildren(
                        layout,
                        cols,
                        this.compactType
                    );

                    this.$set(currentLayouts, breakpoint, layout);

                    this.$emit('layout-init', {layout, layouts: currentLayouts, cols: cols, breakpoint});

                }
            },
            generateLayout(mode = true) {
                if (this.inited && this.layouts instanceof Object && this.ready && !this.disabled) {

                    let currentLayouts = _.cloneDeep(this.layouts);
                    let currentCols = this.cols;
                    let currentBreakpoints = _.cloneDeep(this.breakpoints)

                    const breakpoint = getBreakpointFromWidth(currentBreakpoints, this.containerWidth);
                    const cols = getColsFromBreakpoint(breakpoint, this.colsAll);

                    if (this.breakpoint === breakpoint) {
                    } else {
                        this.onWidthChange(this.containerWidth);
                    }

                    let layout = findOrGenerateResponsiveLayout(
                        currentLayouts,
                        currentBreakpoints,
                        breakpoint,
                        breakpoint,
                        cols,
                        this.compactType
                    );

                    layout = synchronizeLayoutWithChildren(
                        layout,
                        cols,
                        this.compactType
                    );

                    let filtered;
                    filtered = layout.map( (item) => { return { x: item.x, y: item.y, w: item.w, h: item.h, i: item.i }})

                    this.$set(currentLayouts, breakpoint, filtered);

                    this.$emit('layout-generated', {filtered, layouts: currentLayouts, cols: cols, breakpoint, mode: mode});

                }
            },

            synchronizeLayout(mode = true) {
                this.makeSynchronization().then( ({layout, layouts})=> {
                    this.$emit('layout-synchronize', { layout: layout , layouts: layouts});
                }).catch( err => {
                    this.$emit('layout-synchronize-failed');
                })
            },
            makeSynchronization() {
                return new Promise( (resolve) => {
                    let currentLayouts = _.cloneDeep(this.layouts);

                    let newLayout = synchronizeLayoutWithChildren(
                        currentLayouts[this.breakpoint],
                        this.cols,
                        this.compactType
                    );

                    let filtered;
                    filtered = newLayout.map( (item) => { return { x: item.x, y: item.y, w: item.w, h: item.h, i: item.i }})

                    this.$set(currentLayouts, this.breakpoint, filtered);

                    resolve({layout: filtered, layouts: currentLayouts});
                })
            },
            onWidthChange(width) {
                this.containerWidth = width;
                let currentLayouts = _.cloneDeep(this.layouts);

                let breakpoints = _.cloneDeep(this.breakpoints)
                const { cols, colsAll } = this;
                const newBreakpoint = getBreakpointFromWidth(this.breakpoints, this.containerWidth);

                const lastBreakpoint = this.breakpoint;
                const newCols = getColsFromBreakpoint(newBreakpoint, colsAll);

                if (
                    lastBreakpoint !== newBreakpoint ||
                    this.breakpoints !== breakpoints ||
                    cols !== newCols
                ) {
                    if (!(lastBreakpoint in currentLayouts))
                        this.$set(currentLayouts, lastBreakpoint, cloneLayout(this.currentLayout));

                    let currentLayout = findOrGenerateResponsiveLayout(
                        currentLayouts,
                        breakpoints,
                        newBreakpoint,
                        lastBreakpoint,
                        newCols,
                        this.compactType
                    );

                    currentLayout = synchronizeLayoutWithChildren(
                        currentLayout,
                        newCols,
                        this.compactType
                    );

                    this.$set(currentLayouts, newBreakpoint,  currentLayout);

                    this.$emit('breakpoint-change',{breakpoint: newBreakpoint});
                    this.$emit('layout-change',{layout: _.cloneDeep(currentLayout), layouts: _.cloneDeep(currentLayouts), breakpoint: newBreakpoint});
                    this.$emit('width-change', {width: width, cols: newCols});

                } else {
                    this.$emit('width-change', {width: width, cols: newCols})
                }


            },
            updateItemsHeight(mode = true) {
                this.onUpdateItemsHeight(mode).then( (response) => {
                    return true;
                }).catch(err => {
                    return false;
                })
            },
            onUpdateItemsHeight(mode) {
                return new Promise( (resolve) => {
                    let currentLayout = _.cloneDeep(this.layouts[this.breakpoint]);
                    let currentLayouts = _.cloneDeep(this.layouts);

                    let itemsHeightUpdated = currentLayout ? currentLayout.length : -1;

                    if (itemsHeightUpdated <= 0) {
                        this.$emit('layout-height-not-updated');
                        reject(false);
                    }

                    for ( let i =0; i < itemsHeightUpdated; i++) {
                        const id = currentLayout[i].i;

                        eventBus.$emit('updateItemsHeight', id);

                        eventBus.$once('updateHeight'+ id, (id, w, h) => {
                            if (this.queueJobs) {
                                this.jobsQueued.push({i: id, w: w, h: h, job: 'updateHeight'});
                                itemsHeightUpdated--;
                            } else {
                                this.resizeItem(id, w, h).then( ({layout, oldLayout, layouts}) => {
                                    this.$emit('layout-item-height-updated', {layouts: _.cloneDeep(layouts), layout: _.cloneDeep(layout)});
                                    itemsHeightUpdated--;
                                });
                            }
                            if (itemsHeightUpdated === 0) {
                                if (this.queueJobs) {
                                    this.jobsQueued.push({job: 'heightUpdated', mode: mode});
                                } else {
                                    this.$emit('layout-height-updated');
                                }
                                resolve(true);
                            }
                        });
                    }

                })
            },
            resizeAllItems(mode = false, cols = false) {
                this.onResizeAllItems(mode, cols).then((response) => {
                    return true;
                }).catch(err => {
                    return false;
                })
            },

            onResizeAllItems(mode = false, cols = false) {
                return new Promise( (resolve) => {
                    let currentLayout = _.cloneDeep(this.layouts[this.breakpoint]);
                    let currentLayouts = _.cloneDeep(this.layouts);

                    let itemsResized = currentLayout ? currentLayout.length : -1;

                    if (itemsResized <= 0) {
                        this.$emit('layout-not-resized');
                        reject(false);
                    }

                    for ( let i =0; i < itemsResized; i++) {
                        const id = currentLayout[i].i;

                        if ( mode === true && cols === false) {
                            eventBus.$emit('resizeItems', this.cols, id);
                        } else if ( mode === false && cols === false) {
                            eventBus.$emit('resizeItems', null, id);
                        } else if (mode === false && cols) {
                            eventBus.$emit('resizeItems', cols, id);
                        }

                        eventBus.$once('resizeItem'+ id, (id, x, y, w, h, compactType) => {
                            if (this.queueJobs) {
                                this.jobsQueued.push({i: id, x: x, y: y, w: w, h: h, compactType: compactType, job: 'resizeItem'});
                                itemsResized--;
                            } else {
                                this.moveItem(id, x, y, compactType).then( ({layout}) => {
                                    this.resizeItem(id, w, h, layout).then(({layout, layouts}) => {
                                        this.$emit('layout-item-resized', {layouts: _.cloneDeep(layouts), layout: _.cloneDeep(layout)});
                                        itemsResized--;
                                    });
                                });
                            }
                            if (itemsResized === 0) {
                                if (this.queueJobs) {
                                    this.jobsQueued.push({job: 'layoutResized'});
                                } else {
                                    this.$emit('layout-resized');
                                }
                                resolve(true);
                            }
                        });
                    }

                })
            },

            onHeightUpdate(id, w, h) {
                let currentLayout = _.cloneDeep(this.layouts[this.breakpoint]);
                const index = currentLayout.findIndex(item => item.i === id)
                if (index !== -1) {

                    if (this.queueJobs) {
                        this.jobsQueued.push({i: id, w: w, h: h, job: 'updateHeight'});
                    } else {
                        this.resizeItem(id, w, h).then( ({layout, oldLayout, layouts}) => {
                            this.$emit('layout-item-height-updated', {layouts: _.cloneDeep(layouts), layout: _.cloneDeep(layout)});
                        });
                    }

                }
            },

            onLayoutMaybeChanged(newLayout, oldLayout, layouts, mode = false) {
                if (!oldLayout) oldLayout = layouts[this.breakpoint];
                if (!_.isEqual(oldLayout, newLayout) || mode) {
                    const newBreakpoint = getBreakpointFromWidth(this.breakpoints, this.containerWidth);
                    this.$emit('layout-update', {layout: newLayout, layouts: layouts, breakpoint: newBreakpoint});
                }
            },

            resizeItem(i, w, h, layout = this.layouts[this.breakpoint]) {
                return new Promise( (resolve) => {

                    let newLayout = _.cloneDeep(layout);
                    let newLayouts = _.cloneDeep(Object.assign({}, this.layouts, {[this.breakpoint] : layout}));
                    const oldLayout = _.cloneDeep(layout);

                    const { cols, preventCollision } = this;
                    const l = getLayoutItem(newLayout, i);

                    let hasCollisions;
                    if (preventCollision) {
                        const collisions = getAllCollisions(newLayout, { ...l, w, h }).filter((layoutItem) => layoutItem.i !== l.i);
                        hasCollisions = collisions.length > 0;

                        // If we're colliding, we need adjust the placeholder.
                        if (hasCollisions) {
                            // adjust w && h to maximum allowed space
                            let leastX = Infinity, leastY = Infinity;
                            collisions.forEach((layoutItem) => {
                                if (layoutItem.x > l.x) leastX = Math.min(leastX, layoutItem.x);
                                if (layoutItem.y > l.y) leastY = Math.min(leastY, layoutItem.y);
                            });

                            if (Number.isFinite(leastX)) l.w = leastX - l.x;
                            if (Number.isFinite(leastY)) l.h = leastY - l.y;
                        }
                    }

                    if (!hasCollisions) {
                        // Set new width and height.
                        l.w = w;
                        l.h = h;
                    }

                    newLayout = compact(newLayout, this.compactType, cols);
                    const layouts = Object.assign({}, newLayouts, {[this.breakpoint] : newLayout});


                    resolve({layout: _.cloneDeep(newLayout), oldLayout: oldLayout, layouts: _.cloneDeep(layouts)});
                })
            },

            moveItem(i, x, y, compactType = this.compactType, layout = this.layouts[this.breakpoint]) {
                return new Promise( (resolve) => {
                    let newLayout = _.cloneDeep(layout);
                    let newLayouts = _.cloneDeep(Object.assign({}, this.layouts, {[this.breakpoint] : layout}));
                    const oldLayout = _.cloneDeep(layout);
                    let { cols } = this;
                    let l = getLayoutItem(newLayout, i);
                    if (!l) return;

                    this.isDragging = true;

                    newLayout = moveElement(
                        newLayout,
                        l,
                        x,
                        y,
                        false,
                        this.preventCollision,
                        compactType,
                        cols
                    );

                    newLayout = compact(newLayout, compactType, cols);

                    this.isDragging = false;
                    const layouts = Object.assign({}, newLayouts, {[this.breakpoint] : newLayout});

                    resolve({layout: _.cloneDeep(newLayout), oldLayout: oldLayout, layouts: _.cloneDeep(layouts)});
                })
            },

            onDragStart (element, i, x, y, { e, node, newPosition }) {

                let layout = _.cloneDeep(this.layouts[this.breakpoint])

                let l = getLayoutItem(layout, i);
                if (!l) return;


                this.oldDragItem = cloneLayoutItem(l);
                this.oldLayout = _.cloneDeep(layout);
                this.currentLayout = _.cloneDeep(layout);


                this.$emit('onDragStart', layout, l, l, null, e, node);
            },


            onDrag (element, i, x, y, { e, node, newPosition }) {

                const { oldDragItem } = this;
                let { currentLayout } = this;
                let newLayout = _.cloneDeep(currentLayout);
                const { cols } = this;
                let l = getLayoutItem(newLayout, i);
                if (!l) return;

                // Create placeholder (display only)
                this.placeholder = {
                    w: l.w,
                    h: l.h,
                    x: l.x,
                    y: l.y,
                    i: i
                };

                this.isDragging = true;

                // Move the element to the dragged location.
                const isUserAction = true;
                newLayout = moveElement(
                    newLayout,
                    l,
                    x,
                    y,
                    isUserAction,
                    this.preventCollision,
                    this.compactType,
                    cols
                );

                newLayout = compact(newLayout, this.compactType, cols);

                this.$emit('onDrag', newLayout, oldDragItem, l, this.placeholder, e, node);

                const { oldLayout } = this;

                this.currentLayout = _.cloneDeep(newLayout);

                const layouts = Object.assign({}, this.layouts, {[this.breakpoint] : newLayout});

                this.activeDrag = this.placeholder;

                this.onLayoutMaybeChanged(newLayout, oldLayout, layouts);
            },


            onDragStop (element, i, x, y, { e, node, newPosition }) {
                const { oldDragItem } = this;
                let { currentLayout } = this;
                let newLayout = _.cloneDeep(currentLayout);
                const { cols, preventCollision } = this;
                const l = getLayoutItem(newLayout, i);
                if (!l) return;

                // Move the element here
                const isUserAction = true;
                newLayout = moveElement(
                    newLayout,
                    l,
                    x,
                    y,
                    isUserAction,
                    preventCollision,
                    this.compactType,
                    cols
                );

                // Set state
                const layout = compact(newLayout, this.compactType, cols);
                const { oldLayout } = this;

                this.activeDrag = null;
                this.currentLayout = null;


                const layouts = Object.assign({}, this.layouts, {[this.breakpoint] : layout});

                this.oldDragItem = null;
                this.oldLayout = null;
                this.isDragging = false;

                this.$emit('onDragStop', layout, oldDragItem, l, null, e, node);

                this.onLayoutMaybeChanged(layout, oldLayout, layouts);
            },

            onResizeStart (element, i, w, h, { e, node }) {
                let layout = _.cloneDeep(this.layouts[this.breakpoint]);
                let l = getLayoutItem(layout, i);
                if (!l) return;

                this.oldResizeItem = cloneLayoutItem(l);
                this.oldLayout = _.cloneDeep(layout);
                this.currentLayout = _.cloneDeep(layout);

                this.$emit('onResizeStart', layout, l, l, null, e, node);
            },

            onResize (element, i, w, h, { e, node }) {

                const { currentLayout, oldResizeItem } = this;
                let layout = _.cloneDeep(currentLayout);
                const { cols, preventCollision } = this;
                const l = getLayoutItem(layout, i);
                if (!l) return;

                // Something like quad tree should be used
                // to find collisions faster
                let hasCollisions;
                if (preventCollision) {
                    const collisions = getAllCollisions(layout, { ...l, w, h }).filter((layoutItem) => layoutItem.i !== l.i);
                    hasCollisions = collisions.length > 0;

                    // If we're colliding, we need adjust the placeholder.
                    if (hasCollisions) {
                        // adjust w && h to maximum allowed space
                        let leastX = Infinity, leastY = Infinity;
                        collisions.forEach((layoutItem) => {
                            if (layoutItem.x > l.x) leastX = Math.min(leastX, layoutItem.x);
                            if (layoutItem.y > l.y) leastY = Math.min(leastY, layoutItem.y);
                        });

                        if (Number.isFinite(leastX)) l.w = leastX - l.x;
                        if (Number.isFinite(leastY)) l.h = leastY - l.y;
                    }
                }

                if (!hasCollisions) {
                    // Set new width and height.
                    l.w = w;
                    l.h = h;
                }

                this.isResizing = true;
                // Create placeholder element (display only)
                this.placeholder = {
                    w: l.w,
                    h: l.h,
                    x: l.x,
                    y: l.y,
                    static: true,
                    i: i
                };

                this.$emit('onResize', layout, oldResizeItem, l, this.placeholder, e, node);

                const { oldLayout } = this;

                let newLayout = compact(layout, this.compactType, cols);

                this.currentLayout = _.cloneDeep(newLayout);

                this.activeDrag = this.placeholder;

                const layouts = Object.assign({}, this.layouts, {[this.breakpoint] : newLayout});

                this.onLayoutMaybeChanged(newLayout, oldLayout, layouts);

            },

            onResizeStop (element, i, w, h, { e, node }) {

                const { currentLayout, oldResizeItem } = this;
                let layout = _.cloneDeep(currentLayout);
                const { cols } = this;
                let l = getLayoutItem(layout, i);

                // Set state
                const newLayout = compact(layout, this.compactType, cols);

                this.$emit('onResizeStop', newLayout, oldResizeItem, l, null, e, node);

                const { oldLayout } = this;

                this.isResizing = false;
                this.activeDrag = null;
                this.currentLayout = null;

                this.oldResizeItem = null;
                this.oldLayout = null;

                this.onLayoutMaybeChanged(newLayout, oldLayout, true);
            },
        },
        beforeDestroy() {
            eventBus.$off('onDragStart', this.onDragStart);
            eventBus.$off('onDrag', this.onDrag);
            eventBus.$off('onDragStop', this.onDragStop);
            eventBus.$off('onResizeStart', this.onResizeStart);
            eventBus.$off('onResize', this.onResize);
            eventBus.$off('onResizeStop', this.onResizeStop);
            eventBus.$off('updateHeight', this.onHeightUpdate);

            this.$off('layout-ready', this.readyLayout);
            this.$off('width-init', this.widthInited);
        }
    }
</script>
