<template>
    <div :class="classes">
        <div :class="classes2">
            <div class="altchat-comment" :style="styl">
                <div class="altchat-blocks" >
                    <div class="altchat-block altchat-block-paragraph">
                        <p  v-html="text"></p>
                        <!-- <altchat-image v-if="image != null" 
                            :img="image.img" 
                            :href="image.href" 
                            :beforeImgText="image.beforeImgText" 
                            :afterImageText="image.afterImageText">
                        </altchat-image> -->
                        <div v-if="buttons && buttons.length > 0" class="altchat-bot-buttons">
                            <altchat-button 
                                v-for="button in buttons" 
                                :caption="button.caption" 
                                :classes="button.classes">
                            </altchat-button>
                        </div> 
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>

import BeesenderChatButton from './BeesenderChatButton.vue';

export default {
    props: ["text", "buttons", "classes", "classes2", "image", "aftertext", "styl"],

    components: {
        'altchat-button' : BeesenderChatButton
    },

    data () {
        return {
            
        }
    }
}
</script>

<style>

</style>
