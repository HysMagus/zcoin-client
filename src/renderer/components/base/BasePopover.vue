<template>
    <v-popover
        v-bind="$attrs"
        class="popover"
        :popover-class="getPopoverClass"
        :offset="offset"
        :handle-resize="true"
        :auto-hide="this.$attrs['auto-hide'] !== undefined ? this.$attrs['auto-hide'] : false"
        v-on="$listeners"
    >
        <!-- This will be the popover target (for the events and position) -->
        <slot name="target" />

        <template slot="popover">
            <div
                ref="container"
                class="content-wrapper"
            >
                <slot
                    name="content"
                    class="popover-content"
                    @reflow="reflow"
                />
            </div>
        </template>
    </v-popover>
</template>

<script>
import smoothReflow from 'vue-smooth-reflow'
import { mapGetters } from 'vuex'

export default {
    name: 'BasePopover',
    mixins: [
        smoothReflow
    ],
    inheritAttrs: false,

    props: {
        eventBusName: {
            type: String,
            default: 'popover'
        },
        offset: {
            type: [Number, String],
            default: 12
        }
    },

    data () {
        return {
            msg: ''
        }
    },

    computed: {
        ...mapGetters({
            hasOpenOverlay: 'App/hasOpenOverlay'
        }),

        getPopoverClass () {
            const parent = this.$attrs['popover-class'] || ''
            return parent ? [parent] : []
        }
    },

    mounted () {
        this.reflow()
    },

    beforeDestroy () {
        this.$unsmoothReflow({
            el: this.$refs.container
        })
    },

    methods: {
        reflow () {
            this.$smoothReflow({
                el: this.$refs.container,
                transitionEvent: {
                    selector: '.popover-content',
                }
            })
        }
    }
}
</script>

<style lang="scss" scoped>
    .content-wrapper {
        position: relative;
        max-width: 40rem;
    }
</style>
