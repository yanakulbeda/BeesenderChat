<template>
  <button :class="classes" @click="onbuttonclick(caption)"> {{ caption }} </button>
</template>

<script>

export default {

    props: {
        caption : {
            type: String,
            default: ''
        },

        classes : {
            type : String,
            default : ''
        }
    },

    data() {    
        return {

        }
    },

    methods: {
        onbuttonclick: function () {
            [].forEach.call(document.getElementsByClassName('altchat-bot-buttons'), function(element) {
                element.remove();
            });

            this.$parent.$parent.$parent.sendMsgToServerMethod(this.caption);
        }
    }
}
</script>

<style>

</style>
