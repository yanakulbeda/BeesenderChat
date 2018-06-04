<template>
    <div @click="show" id="altchat-widget-startbutton" data-altchat-button-cont="" class="beesender-widget-button-wrapper beesender-widget-button-position-bottom-right beesender-widget-button-visible ">
        <div data-altchat-hello-cont="" class="beesender-widget-button-popup" v-bind:style="{ 'background-color': themeStyle }" >
        <span data-beesender-hello-btn-hide="" class="beesender-widget-button-popup-btn-hide"></span>
        <div class="beesender-widget-button-popup-inner">
            <div class="beesender-widget-button-popup-image">
                <span data-beesender-hello-icon="" class="beesender-widget-button-popup-image-item"></span>
            </div>
            <div class="beesender-widget-button-popup-content">
                <div data-beesender-hello-name="" class="beesender-widget-button-popup-name"></div>
                <div data-beesender-hello-text="" class="beesender-widget-button-popup-description"></div>
            </div>
        </div>
        <div class="beesender-widget-button-popup-triangle" v-bind:style="{ 'background-color': themeStyle }"></div>
        </div>
        <div data-altchat-button-block-button="" class="beesender-widget-button-inner-container">
            <div data-altchat-button-block-border="" class="beesender-widget-button-inner-mask" style="background: #3AD0FF;"></div>
            <div class="beesender-widget-button-block">
                <div data-altchat-button-pulse="" class="beesender-widget-button-pulse beesender-widget-button-pulse-animate" v-bind:style="{ 'background-color': themeStyle }"></div>
                <div data-altchat-button-block-inner="" class="beesender-widget-button-inner-block" v-bind:style="{ 'background-color': themeStyle }">
                    <div class="beesender-widget-button-icon-container">
                        <div  data-altchat-button-icon="openline" class="beesender-widget-button-inner-item beesender-widget-button-icon-animation">
                            <img width="39" height="31" viewbox="0 0 28 29" src="https://v2.messenger.beesender.com/images/faq-icon.png" class="altchat-button-icon altchat-button-icon-active">
                        </div>
                    </div>
                    <div class="beesender-widget-button-inner-item beesender-widget-button-close">
                        <svg class="beesender-widget-button-icon beesender-widget-button-close-item" xmlns="https://www.w3.org/2000/svg" width="29" height="29" viewBox="0 0 29 29"><path fill="#FFF" fill-rule="evenodd" d="M18.866 14.45l9.58-9.582L24.03.448l-9.587 9.58L4.873.447.455 4.866l9.575 9.587-9.583 9.57 4.418 4.42 9.58-9.577 9.58 9.58 4.42-4.42"></path></svg>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
    export default {
        data () {
            return {
                themeStyle: this.$parent._props.themeColor
            };
        },

        methods: {

            show () {
                this.$parent.isVisible = true;
            }

        }
    }
</script>

<style>

</style>
