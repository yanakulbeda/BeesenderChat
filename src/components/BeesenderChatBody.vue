<template>
    <div class="altchat-conversation-body-container"> <div class="altchat-conversation-backgrounds">
        <div class="altchat-conversation-background altchat-conversation-background-1"></div>
    </div>
    <div class="altchat-conversation-body" style="-webkit-transform: translateY(-227px); transform: translateY(-227px); bottom: -227px;">
        <div class="altchat-conversation-body-profile" style="background-color: #21293d">
        <div class="altchat-conversation-profile">
            <div class="altchat-team-profile altchat-team-profile-collapsed altchat-team-profile-office-hours" style="margin-top: 305px;">
                <div class="altchat-team-profile-compact" style="opacity: 1;left: 0px;top: 227px;">
                    <div class="altchat-team-profile-compact-contents">
                    <div class="altchat-team-profile-compact-avatar-container">
                        <div class="altchat-team-profile-compact-avatar" style="z-index: 3; opacity: 1; margin-left: 0px;">
                            <div class="altchat-avatar">
                                <img v-bind:src="this.headerImageSrc">
                            </div>
                        </div>
                    </div>
                    <div class="interco#altchat-container .altchat-team-profile-collapsed, #altchat-container .altchat-team-profile-collapsed *m-team-profile-compact-body">
                        <div class="altchat-team-profile-compact-team-name">
                            {{title}}
                        </div>
                        <div class="altchat-team-profile-compact-response-delay">
                            <span class="altchat-team-profile-response-delay-text">
                                {{subtitle}}
                            </span>
                        </div>
                    </div>
                    </div>
                </div>
            </div>
        </div>
        </div>
        <div id="altchat-main-feed" class="altchat-conversation-body-parts" style="bottom: 55px; top: 302px;">
        <div class="altchat-conversation-body-parts-wrapper">
            <div class="altchat-conversation-parts altchat-conversation-parts-scrolled" style="transform: translateY(0px);">
                <span>
                    <altchat-message v-for="msg in messages" :text="msg.text" :buttons="msg.buttons" :styl="msg.style" :classes="msg.classes" :classes2="msg.classes2" :image="msg.image" :aftertext="msg.aftertext"></altchat-message>
                </span>
                    </div>
                </div>
            </div>
    </div>
    <span>
        <div class = "" ></div>
    </span>
    </div>
</template>

<script>

import BeesenderChatMessage from './BeesenderChatMessage.vue';

export default {
    
    props: ["messages"],

    components: {
        'altchat-message' : BeesenderChatMessage
    },

    data() {
        return {
            title : this.$parent._props.title,
            subtitle : '',
            headerImageSrc : this.$parent._props.headerImageSrc
        }
    }
}

</script>

<style>

</style>
