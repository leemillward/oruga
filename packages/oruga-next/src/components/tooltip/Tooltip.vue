<template>
    <span
        ref="tooltip"
        :class="rootClasses">
        <transition :name="newAnimation">
            <div
                v-show="active && (isActive || always)"
                ref="content"
                :class="contentClasses">
                <span :class="arrowClasses"></span>
                <template v-if="label">{{ label }}</template>
                <template v-else-if="$slots.default">
                    <slot name="content" />
                </template>
            </div>
        </transition>
        <div
            ref="trigger"
            :class="triggerClasses"
            :style="triggerStyle"
            @click="onClick"
            @contextmenu="onContextMenu"
            @mouseenter="onHover"
            @focus.capture="onFocus"
            @blur.capture="close"
            @mouseleave="close">
            <slot ref="slot" />
        </div>
    </span>
</template>

<script lang="ts">
import { getOptions } from '../../utils/config'
import BaseComponentMixin from '../../utils/BaseComponentMixin'
import { createAbsoluteElement, removeElement, getValueByPath } from '../../utils/helpers'
import { defineComponent } from 'vue'

/**
 * Display a brief helper text to your user
 * @displayName Tooltip
 * @example ./examples/Tooltip.md
 * @style _tooltip.scss
 */
export default defineComponent({
    name: 'OTooltip',
    mixins: [BaseComponentMixin],
    configField: 'tooltip',
    props: {
        /** Whether tooltip is active or not, use the .sync modifier (Vue 2.x) or v-model:active (Vue 3.x) to make it two-way binding */
        active: {
            type: Boolean,
            default: true
        },
        /** Tooltip text */
        label: String,
        /** Tooltip delay before it appears (number in ms) */
        delay: Number,
        /**
         * Tooltip position in relation to the element
         * @values top, bottom, left, right
         */
        position: {
            type: String,
            default: () => { return getValueByPath(getOptions(), 'tooltip.position', 'top') },
            validator: (value: string) => {
                return [
                    'top',
                    'bottom',
                    'left',
                    'right'
                ].indexOf(value) > -1
            }
        },
        /**
         * Tooltip trigger events
         * @values hover, click, focus, contextmenu
         */
        triggers: {
            type: Array,
            default: () => { return getValueByPath(getOptions(), 'tooltip.triggers', ['hover']) }
        },
        /** Tooltip will be always active */
        always: Boolean,
        /** Tooltip will have an animation */
        animated: {
            type: Boolean,
            default: true
        },
        /** Tooltip default animation */
        animation: {
            type: String,
            default: () => { return getValueByPath(getOptions(), 'tooltip.animation', 'fade') }
        },
        /**
         * Tooltip auto close options
         * @values true, false, 'inside', 'outside'
         */
        autoClose: {
            type: [Array, Boolean],
            default: true
        },
        /** Tooltip will be multilined */
        multiline: Boolean,
        /** Append tooltip content to body */
        appendToBody: Boolean,
        /**
        * Color of the tooltip
        * @values primary, info, success, warning, danger, and any other custom color
        */
        variant: [String, Function, Array],
        rootClass: [String, Function, Array],
        contentClass: [String, Function, Array],
        orderClass: [String, Function, Array],
        triggerClass: [String, Function, Array],
        multilineClass: [String, Function, Array],
        alwaysClass: [String, Function, Array],
        variantClass: [String, Function, Array],
        arrowClass: [String, Function, Array],
        arrowOrderClass: [String, Function, Array]
    },
    data() {
        return {
            isActive: false,
            triggerStyle: {},
            bodyEl: undefined // Used to append to body
        }
    },
    computed: {
        rootClasses() {
            return [
                this.computedClass('rootClass', 'o-tip')
            ]
        },
        triggerClasses() {
            return [
                this.computedClass('triggerClass', 'o-tip__trigger'),
            ]
        },
        arrowClasses() {
            return [
                this.computedClass('arrowClass', 'o-tip__arrow'),
                { [this.computedClass('arrowOrderClass', 'o-tip__arrow--', this.position)]: this.position },
                { [this.computedClass('variantArrowClass', 'o-tip__arrow--', this.variant)]: this.variant },
            ]
        },
        contentClasses() {
            return [
                this.computedClass('contentClass', 'o-tip__content'),
                { [this.computedClass('orderClass', 'o-tip__content--', this.position)]: this.position },
                { [this.computedClass('variantClass', 'o-tip__content--', this.variant)]: this.variant },
                { [this.computedClass('multilineClass', 'o-tip__content--multiline')]: this.multiline },
                { [this.computedClass('alwaysClass', 'o-tip__content--always')]: this.always }
            ]
        },
        newAnimation() {
            return this.animated ? this.animation : undefined
        }
    },
    watch: {
        isActive(value) {
            if (value && this.appendToBody) {
                this.updateAppendToBody()
            }
        }
    },
    methods: {
        updateAppendToBody() {
            const tooltip = this.$refs.tooltip
            const trigger = this.$refs.trigger
            if (tooltip && trigger) {
                // update wrapper tooltip
                const tooltipEl = this.$data.bodyEl.children[0]
                tooltipEl.classList.forEach((item) => tooltipEl.classList.remove(...item.split(' ')))
                if (this.$vnode && this.$vnode.data && this.$vnode.data.staticClass) {
                    tooltipEl.classList.add(this.$vnode.data.staticClass)
                }
                this.rootClasses.forEach((item) => {
                    if (typeof item === 'object') {
                        Object.keys(item).filter(key => !!item[key]).forEach(
                            key => tooltipEl.classList.add(key))
                    } else {
                        tooltipEl.classList.add(...item.split(' '))
                    }
                })
                tooltipEl.style.width = `${trigger.clientWidth}px`
                tooltipEl.style.height = `${trigger.clientHeight}px`
                const rect = trigger.getBoundingClientRect()
                const top = rect.top + window.scrollY
                const left = rect.left + window.scrollX
                const wrapper = this.$data.bodyEl
                wrapper.style.position = 'absolute'
                wrapper.style.top = `${top}px`
                wrapper.style.left = `${left}px`
                wrapper.style.zIndex = this.isActive || this.always ? '99' : '-1'
                this.triggerStyle = { zIndex: this.isActive || this.always ? '100' : undefined }
            }
        },
        onClick() {
            if (this.triggers.indexOf('click') < 0) return
            // if not active, toggle after clickOutside event
            // this fixes toggling programmatic
            this.$nextTick(() => {
                setTimeout(() => this.open())
            })
        },
        onHover() {
            if (this.triggers.indexOf('hover') < 0) return
            this.open()
        },
        onFocus() {
            if (this.triggers.indexOf('focus') < 0) return
            this.open()
        },
        onContextMenu(event) {
            if (this.triggers.indexOf('contextmenu') < 0) return
            event.preventDefault()
            this.open()
        },
        open() {
            if (this.delay) {
                this.timer = setTimeout(() => {
                    this.isActive = true
                    this.timer = null
                }, this.delay)
            } else {
                this.isActive = true
            }
        },
        close() {
            if (typeof this.autoClose === 'boolean') {
                this.isActive = !this.autoClose
            }
            if (this.autoClose && this.timer) clearTimeout(this.timer)
        },
        /**
        * Close tooltip if clicked outside.
        */
        clickedOutside(event) {
            if (this.isActive) {
                if (Array.isArray(this.autoClose)) {
                    if (this.autoClose.indexOf('outside') >= 0) {
                        if (!this.isInWhiteList(event.target)) this.isActive = false
                    }
                    if (this.autoClose.indexOf('inside') >= 0) {
                        if (this.isInWhiteList(event.target)) this.isActive = false
                    }
                }
            }
        },
        /**
         * Keypress event that is bound to the document
         */
        keyPress({ key }) {
            if (this.isActive && (key === 'Escape' || key === 'Esc')) {
                if (Array.isArray(this.autoClose)) {
                    if (this.autoClose.indexOf('escape') >= 0) this.isActive = false
                }
            }
        },
        /**
        * White-listed items to not close when clicked.
        */
        isInWhiteList(el) {
            if (el === this.$refs.content) return true
            // All chidren from content
            if (this.$refs.content !== undefined) {
                const children = this.$refs.content.querySelectorAll('*')
                for (const child of children) {
                    if (el === child) {
                        return true
                    }
                }
            }
            return false
        }
    },
    mounted() {
        if (this.appendToBody) {
            this.$data.bodyEl = createAbsoluteElement(this.$refs.content)
            this.updateAppendToBody()
        }
    },
    created() {
        if (typeof window !== 'undefined') {
            document.addEventListener('click', this.clickedOutside)
            document.addEventListener('keyup', this.keyPress)
        }
    },
    beforeUnmount() {
        if (typeof window !== 'undefined') {
            document.removeEventListener('click', this.clickedOutside)
            document.removeEventListener('keyup', this.keyPress)
        }
        if (this.appendToBody) {
            removeElement(this.$data.bodyEl)
        }
    }
})
</script>
