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
            makeUpdateOnInit: {
                required: false,
                type: Boolean,
                default: true
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
                itemsResized: 0

            }
        },
        created() {
            eventBus.$on('onDragStart', this.onDragStart);
            eventBus.$on('onDrag', this.onDrag);
            eventBus.$on('onDragStop', this.onDragStop);
            eventBus.$on('onResizeStart', this.onResizeStart);
            eventBus.$on('onResize', this.onResize);
            eventBus.$on('onResizeStop', this.onResizeStop);
            eventBus.$on('onResizeItem', this.onResizeItem);
            eventBus.$on('onMoveItem', this.onMoveItem);

            this.$on('layout-ready', this.readyLayout);
            this.$on('width-init', this.widthInited);
        },

        watch: {
            ready(val) {
                if (val) {
                    this.$emit('layout-ready', {width : this.containerWidth});
                }
            }
        },

        methods: {
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

                    let currentLayouts = {...this.layouts};
                    let currentCols = this.cols;
                    let currentBreakpoints = {...this.breakpoints}

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
                    // Provided to make sure that components are re-rendered
                    // Sometimes event handlers makes the errors

                    if (this.makeUpdateOnInit) {
                        this.$nextTick( ()=> {
                            this.onUpdateItemsHeight(mode, currentLayouts, layout).then( ({layout, layouts}) => {
                                this.$emit('layout-init', {layout, layouts, cols, breakpoint});
                            }).catch(err => {
                                this.$emit('layout-init-failed');
                            })
                        })
                    } else {
                        this.$emit('layout-init', {layout, layouts: currentLayouts, cols, breakpoint});
                    }

                }
            },

            synchronizeLayout(mode = true) {
                this.makeSynchronization().then( ({layout, layouts})=> {
                    this.$nextTick( ()=> {
                        this.onUpdateItemsHeight(mode, layout, layouts).then( ({layout, layouts}) => {
                            this.$emit('layout-synchronize', { layout: layout , layouts: layouts});
                        }).catch(err => {
                            this.$emit('layout-synchronize-failed');
                        })
                    })
                })
            },
            makeSynchronization() {
                return new Promise( (resolve) => {
                    let currentLayouts = {...this.layouts};

                    let newLayout = synchronizeLayoutWithChildren(
                        this.currentLayouts[this.breakpoint],
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
                let currentLayouts = {...this.layouts};
                let breakpoints = {... this.breakpoints}
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
                    this.$emit('layout-change',{layout: currentLayout, layouts: currentLayouts, breakpoint: newBreakpoint});
                    this.$emit('width-change', {width: width, cols: newCols});

                } else {
                    this.$emit('width-change', {width: width, cols: newCols})
                }


            },
            updateItemsHeight(mode = true) {
                this.onUpdateItemsHeight(mode).then( ({layout, layouts}) => {
                    this.$emit('layout-height-updated', {layouts: layouts, layout: layout});
                }).catch(err => {
                    this.$emit('layout-height-updated-failed');
                })
            },
            onUpdateItemsHeight(mode, layouts = {...this.layouts}, layout = [...this.layouts[this.breakpoint]]) {
                return new Promise( (resolve) => {
                    let itemsHeightUpdated = layout ? layout.length : -1;
                    let currentLayout = [...layout];

                    for ( let i =0; i < currentLayout.length; i++) {
                        eventBus.$emit('updateItemsHeight', layouts, layout, ({layout, oldLayout, layouts}) => {
                            itemsHeightUpdated--;

                            if (itemsHeightUpdated === 0) {
                                this.$set(layouts, this.breakpoint,  layout);
                                if (mode)
                                    this.onLayoutMaybeChanged(layout, oldLayout, layouts, mode);
                                resolve({layout: layout, layouts: layouts, oldLayout})
                            }
                        });
                    }


                })
            },
            resizeAllItems(mode = false, cols = false) {
                this.onResizeAllItems(mode, cols).then(({layout, layouts, oldLayout}) => {
                    this.$emit('layout-resized', {layouts: layouts, layout: layout});
                })
            },
            onResizeAllItems(mode = false, cols = false) {
                return new Promise( (resolve) => {
                    let currentLayouts = {...this.layouts};
                    let currentLayout = [...this.layouts[this.breakpoint]]
                    let itemsResized = currentLayout ? currentLayout.length : -1;

                    if (itemsResized > 0) {
                        for (let i = 0; i < currentLayout.length; i++) {
                            const item = currentLayout.find(item => item.i === currentLayout[i].i);
                            if ((mode === false) && (cols === false)) {
                                eventBus.$emit('resizeAllItems', null, currentLayout, item.i, ({layout, oldLayout, layouts}) => {

                                    itemsResized--;

                                    currentLayout[i] = layout[i];


                                    if (itemsResized === 0) {
                                        this.$set(currentLayouts, this.breakpoint,  currentLayout);

                                        resolve({layout: currentLayout, layouts: currentLayouts, oldLayout})
                                    }

                                });
                            }

                            if ((mode === true) && (cols === false)) {
                                eventBus.$emit('resizeAllItems', this.cols, currentLayout, item.i,({layout, oldLayout, layouts}) => {

                                    itemsResized--;

                                    currentLayout[i] = layout[i];

                                    if (itemsResized === 0) {
                                        this.$set(currentLayouts, this.breakpoint,  currentLayout);

                                        resolve({layout: currentLayout, layouts: currentLayouts, oldLayout})
                                    }
                                });
                            }

                            if ((cols !== false) && (typeof cols === 'number')) {
                                eventBus.$emit('resizeAllItems', cols, currentLayout, item.i, ({layout, oldLayout, layouts}) => {

                                    itemsResized--;

                                    currentLayout[i] = layout[i];


                                    if (itemsResized === 0) {
                                        this.$set(currentLayouts, this.breakpoint,  currentLayout);

                                        resolve({layout: currentLayout, layouts: currentLayouts, oldLayout})
                                    }
                                });
                            }
                        }
                    }


                });

            },
            onResizeItem(id, w, newH, mode = false, layout = [...this.layouts[this.breakpoint]], callback) {
                const index = layout.findIndex(item => item.i === id)
                if (index !== -1) {
                    this.resizeItem(id, w, newH, layout).then( ({layout, oldLayout, layouts}) => {
                        if (mode === 'resizeAll') {
                            if (callback)
                                callback({layout, oldLayout, layouts})
                        }
                        if (mode === 'updateHeight') {
                            if (callback)
                                callback({layout, oldLayout, layouts})
                        }
                    });
                }
            },

            onLayoutMaybeChanged(newLayout, oldLayout, layouts, mode = false) {
                if (!oldLayout) oldLayout = layouts[this.breakpoint];
                if (!_.isEqual(oldLayout, newLayout) || mode) {
                    const newBreakpoint = getBreakpointFromWidth(this.breakpoints, this.containerWidth);
                    this.currentBreakpoint = newBreakpoint;
                    this.$emit('layout-update', {layout: newLayout, layouts: layouts, breakpoint: newBreakpoint});
                }
            },
            resizeItem(i, w, h, layout) {
                return new Promise( (resolve) => {

                    let currentLayout = [...layout];
                    const oldLayout = [...layout];

                    const { cols, preventCollision } = this;
                    const l = getLayoutItem(currentLayout, i);

                    let hasCollisions;
                    if (preventCollision) {
                        const collisions = getAllCollisions(currentLayout, { ...l, w, h }).filter((layoutItem) => layoutItem.i !== l.i);
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

                    currentLayout = compact(currentLayout, this.compactType, cols);
                    const layouts = Object.assign({}, this.layouts, {[this.breakpoint] : currentLayout})


                    resolve({layout: currentLayout, oldLayout: oldLayout, layouts: layouts});
                })

            },
            onMoveItem(i, x, y, compactType = this.compactType, layout = [...this.layouts[this.breakpoint]], callback) {

                const index = layout.findIndex(item => item.i === i)
                if (index !== -1) {
                    this.moveItem(i, x, y, compactType, layout).then( ({layout, oldLayout, layouts}) => {
                        if (callback) {
                            callback({layout, oldLayout, layouts})
                        }
                    });
                }
            },

            moveItem(i, x, y, compactType = this.compactType, layout) {
                return new Promise( (resolve) => {
                    let { cols } = this;
                    let l = getLayoutItem(layout, i);
                    if (!l) return;

                    let oldLayout = [...layout];

                    this.isDragging = true;

                    layout = moveElement(
                        layout,
                        l,
                        x,
                        y,
                        false,
                        this.preventCollision,
                        compactType,
                        cols
                    );

                    const newLayout = compact(layout, compactType, cols);

                    this.isDragging = false;
                    const layouts = Object.assign({}, this.layouts, {[this.breakpoint] : newLayout});

                    resolve({layout: newLayout, oldLayout: oldLayout, layouts: layouts});
                })
            },

            onDragStart (element, i, x, y, { e, node, newPosition }) {

                let layout = [...this.layouts[this.breakpoint]]

                let l = getLayoutItem(layout, i);
                if (!l) return;


                this.oldDragItem = cloneLayoutItem(l);
                this.oldLayout = [...layout];
                this.currentLayout = [...layout];


                this.$emit('onDragStart', [...layout], l, l, null, e, node);
            },


            onDrag (element, i, x, y, { e, node, newPosition }) {

                const { oldDragItem } = this;
                let { currentLayout } = this;
                let newLayout = [...currentLayout];
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

                this.currentLayout = [...newLayout];

                const layouts = Object.assign({}, this.layouts, {[this.breakpoint] : newLayout});

                this.activeDrag = this.placeholder;

                this.onLayoutMaybeChanged(newLayout, oldLayout, layouts);
            },


            onDragStop (element, i, x, y, { e, node, newPosition }) {
                const { oldDragItem } = this;
                let { currentLayout } = this;
                const { cols, preventCollision } = this;
                const l = getLayoutItem(currentLayout, i);
                if (!l) return;

                // Move the element here
                const isUserAction = true;
                currentLayout = moveElement(
                    currentLayout,
                    l,
                    x,
                    y,
                    isUserAction,
                    preventCollision,
                    this.compactType,
                    cols
                );

                // Set state
                const newLayout = compact(currentLayout, this.compactType, cols);
                const { oldLayout } = this;

                this.activeDrag = null;
                this.currentLayout = null;


                const layouts = Object.assign({}, this.layouts, {[this.breakpoint] : newLayout});

                this.oldDragItem = null;
                this.oldLayout = null;
                this.isDragging = false;

                this.$emit('onDragStop', newLayout, oldDragItem, l, null, e, node);

                this.onLayoutMaybeChanged(newLayout, oldLayout, layouts);
            },

            onResizeStart (element, i, w, h, { e, node }) {
                let layout = [...this.layouts[this.breakpoint]]
                let l = getLayoutItem(layout, i);
                if (!l) return;

                this.oldResizeItem = cloneLayoutItem(l);
                this.oldLayout = [...layout];
                this.currentLayout = [...layout];

                this.$emit('onResizeStart', layout, l, l, null, e, node);
            },

            onResize (element, i, w, h, { e, node }) {

                const { currentLayout, oldResizeItem } = this;
                const { cols, preventCollision } = this;
                const l = getLayoutItem(currentLayout, i);
                if (!l) return;

                // Something like quad tree should be used
                // to find collisions faster
                let hasCollisions;
                if (preventCollision) {
                    const collisions = getAllCollisions(currentLayout, { ...l, w, h }).filter((layoutItem) => layoutItem.i !== l.i);
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

                this.$emit('onResize', currentLayout, oldResizeItem, l, this.placeholder, e, node);

                const { oldLayout } = this;

                this.currentLayout = compact(currentLayout, this.compactType, cols);

                this.activeDrag = this.placeholder;

                const layouts = Object.assign({}, this.layouts, {[this.breakpoint] : this.currentLayout});

                this.onLayoutMaybeChanged([...this.currentLayout], oldLayout, layouts);

            },

            onResizeStop (element, i, w, h, { e, node }) {

                const { currentLayout, oldResizeItem } = this;
                const { cols } = this;
                let l = getLayoutItem(currentLayout, i);

                // Set state
                const newLayout = compact(currentLayout, this.compactType, cols);

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
            eventBus.$off('onMoveItem', this.onMoveItem);
            eventBus.$off('onResizeItem', this.onResizeItem);

            this.$off('layout-ready', this.readyLayout);
            this.$off('width-init', this.widthInited);
        }
    }
</script>
