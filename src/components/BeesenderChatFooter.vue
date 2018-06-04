<template>

    <div class="altchat-conversation-footer" style="heigh: 55px;z-index: 10;height: 90px;">
        <div class="altchat-composer" style="bottom: 25px; ">
            <pre><br></pre>
            <textarea id="altchat-sendbox-textarea" v-model="currentMessage" data-autoresize="" rows="2" :placeholder="chatMessagePlaceholder"></textarea>
            <span></span><span></span>
            <div class="altchat-composer-buttons">
                <!-- <div class="altchat-composer-upload-button">
                    <input type="file" style="display: none;">
                </div> -->
                <button class="altchat-composer-send-button" @click="onsendclick(currentMessage);clearCurrentMessage();" v-bind:style="{ 'background-image' : 'url(' + sendButtonIcon + ')' }"></button>
            </div>
        </div>
        <div class="altchat-footer-description">
            <a href="http://beesender.com/" target="_blank" >{{footerTitle}}</a>
        </div>
    </div>
     
</template>

<script>
export default {
    props: ["onsendclick"],

    data() {
        return {
            currentMessage: '',
            chatMessagePlaceholder: this.$parent._props.chatMessagePlaceholder,
            footerTitle: this.$parent._props.footerTitle,
            sendButtonIcon: this.$parent._props.sendButtonIcon
        }
    },

    methods: {
        clearCurrentMessage: function () {
            this.currentMessage = '';
        }
    }
}
</script>

<style>

</style>
