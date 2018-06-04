<template>

    <div class="altchat-header-buttons">
        <div class="altchat-header-buttons-close altchat-header-buttons-print" title="Распечатать историю переписки">
            <div id="altchat-printbutton" class="altchat-header-buttons-close-contents"></div>
        </div>
        <div class="altchat-header-buttons-close" title="Закрыть чат">
            <div v-on:click="hide" id="altchat-closebutton" class="altchat-header-buttons-close-contents"></div>
        </div>
    </div>

</template>

<script>

export default {


    components: {

    },

    data () {
        return {
           
        }
    },

    methods: {

        hide: function() {
            this.$parent.isVisible = false;
        },

        getCurrentDate: function() {
            var today = new Date();
            var dd = today.getDate();
            var mm = today.getMonth() + 1
            var yyyy = today.getFullYear();

            if (dd < 10) {
                dd = '0' + dd
            }

            if (mm < 10) {
                mm = '0' + mm
            }

            today = dd + '-' + mm + '-' + yyyy;
            return today;
        }
    }
}
</script>

<style>

</style>
