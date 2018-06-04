<template>
    <div id="altchat">
        <div v-show="isVisible" id= "altchat-container-body">
        <div id="altchat-container">
            <div data-reactroot="" class="altchat-messenger"style="width: 350px;height: 100%;box-shadow: rgba(0, 0, 0, 0.3) 0px 0px 59px;z-index: 10;position: fixed;">
                <div class="altchat-messenger-background">
            </div>
            <span>
                <div class="altchat-conversation">
                    <altchat-header></altchat-header>
                    <altchat-body :messages="messages"></altchat-body>
                    <altchat-footer :onsendclick="onSendClick"></altchat-footer> 
                </div>
            </span>
            </div>
        </div>
        </div>
        <altchat-widget-button v-show="!isVisible"></altchat-widget-button>
    </div>
</template>

<script>

import BeesenderChatHeader from './BeesenderChatHeader.vue';
import BeesenderChatWidgetButton from './BeesenderChatWidgetButton.vue';
import BeesenderChatFooter from './BeesenderChatFooter.vue';
import BeesenderChatBody from './BeesenderChatBody.vue';

import signalR from '../js/signalr-client-1.0.0-alpha1-final.min.js';


export default {

    mounted: function() {
        var channelId =this.channelId;
        var clientId = this.getCookie("beesender:" + channelId);

        var hubUrl = 'https://v2.messenger.beesender.com/signalr?channelId=' + channelId + "&clientId=" + clientId;
        

        var httpConnection = new window.signalR.HttpConnection(hubUrl);
        var hubConnection = new window.signalR.HubConnection(httpConnection);
        

        var self = this;
        hubConnection.on('SetCookies', function (key, value) {
            self.setCookie(key, value);
        });

        hubConnection.on('AddMessage', function (msg) {
            self.addMessage(
                self.getClientSimpleMessage(msg.replace(/\n+/g, "\n").trim()))
        });

        hubConnection.on('unBindClientTypingHandler', function () {
            self.unBindClientTypingHandler();
        });

        hubConnection.on('SendTextMessage', function (message) {
            if (!message.Buttons) {
                self.setInputDisabledStatus(false);
                self.clearInputField();
                self.addOperatorMessage(self.getOperatorSimpleMessage(message.Text));
            }
            if (message.Buttons) {

                self.setInputDisabledStatus(true);

                if (message.Buttons.length == 0) {
                    self.setInputDisabledStatus(false);
                    self.clearInputField();
                }
                self.addOperatorMessage(self.getOperatorButtonsMessage(message));
            }
        });

        function httpGetAsync(theUrl, callback)
        {
            var xmlHttp = new XMLHttpRequest();
            xmlHttp.onreadystatechange = function() { 
                if (xmlHttp.readyState == 4 && xmlHttp.status == 200)
                    callback(xmlHttp.responseText);
            }
            xmlHttp.open("GET", theUrl, true); // true for asynchronous 
            xmlHttp.send(null);
        }

        if (self.collectUserInfo) {
            if (hubConnection.connection.connectionState != 2)
                hubConnection.start().then(() => {
                    this.sendMsgToServerMethod = function (content, clientId) {
                        hubConnection.invoke("SendMessage", content, clientId);
                    };
                    this.sendTypingText = function (content) {
                        hubConnection.invoke("SendTypingText", content, this.clientId);
                    };

                    var storeItem = window.localStorage.getItem(this.id);

                    if (storeItem) {
                        var history = JSON.parse(storeItem);
                        if (Math.abs(new Date() - new Date(history.time)) > 1000 * 60 * this.localStorageLifeTime) {
                            window.localStorage.removeItem(this.id);
                        }
                    }

                    storeItem = window.localStorage.getItem(self.id);
                    if (!storeItem) {
                        
                        httpGetAsync('https://api.2ip.ua/geo.json?ip=', function ipInfoCallback(data) {

                            var dataObj = JSON.parse(data);

                            var ipUserInfo = {
                                IP: dataObj.ip,
                                Country: dataObj.country,
                                Region: dataObj.region,
                                City: dataObj.city,
                                Timezone: dataObj.time_zone,

                                Browser: self.GetUserBrowserName(),
                                Platform: navigator.platform,
                                Directory: document.location.pathname,
                                Previous_Site: document.referrer
                            }
                            self.onSendClick('#[UserInfo]#' + JSON.stringify(ipUserInfo));
                            var history = JSON.parse(window.localStorage.getItem(self.id)) || {};
                            self.messages = history.messages;
                            self.rerenderFeed();
                        });

                        /*$.getJSON('https://api.2ip.ua/geo.json?ip=', function ipInfoCallback(data) {
                            var ipUserInfo = {
                                IP: data.ip,
                                Country: data.country,
                                Region: data.region,
                                City: data.city,
                                Timezone: data.time_zone,

                                Browser: self.GetUserBrowserName(),
                                Platform: navigator.platform,
                                Directory: document.location.pathname,
                                Previous_Site: document.referrer
                            }
                            self.onSendClick('#[UserInfo]#' + JSON.stringify(ipUserInfo));
                            var history = JSON.parse(window.localStorage.getItem(self.id)) || {};
                            self.messages = history.messages;
                            self.rerenderFeed();
                        });*/
                    }
                    var history = JSON.parse(window.localStorage.getItem(self.id));
                    if (history) {
                        self.messages = history.messages;
                        self.rerenderFeed();
                    }
                });

                document.getElementById('altchat-sendbox-textarea')
                    .addEventListener('keyup', function (key) {
                        if (!key.shiftKey && key.key === "Enter" && 
                                document.getElementById('altchat-sendbox-textarea').value) {
                            document.getElementsByClassName('altchat-composer-send-button')[0].click();
                        }
                });

                document.getElementById('altchat-sendbox-textarea')
                    .addEventListener('keyup', self.typingHandler);
        }
    },

    props: {
        title: {
            type: String,
            default: 'Beesender'
        },

        footerTitle: {
            type: String,
            default: 'Beesender - чаты и чатботы'
        },

        chatMessagePlaceholder: {
            type: String,
            default: 'Введите сообщение...'
        },

        chatButtonsPlaceholder: {
            type: String,
            default: 'Выберите один из предложенных вариантов'
        },

        cssSrc: {
            type: String,
            default: 'https://v2.messenger.beesender.com/app/app.css'
        },

        widgetButtonSrc: {
            type: String,
            default: 'https://v2.messenger.beesender.com/images/bs2.png'
        },

        headerImageSrc: {
            type: String,
            default: 'http://beesender.com/beesender-icon.png'
        },

        sendButtonIcon: {
            type: String,
            default: 'https://v2.messenger.beesender.com/images/send-button.png'
        },

        themeColor: {
            type: String,
            default: '#21293d'
        },

        right: {
            type: String,
            default: '30px'
        },

        bottom: {
            type: String,
            default: '30px'
        },

        collectUserInfo: {
            type: Boolean,
            default: true
        },

        localStorageLifeTime: {
            type: Number,
            default: 30
        },

        disableInputOnButtonMessages: {
            type: Boolean,
            default: false
        },

        clientId : {
            type: String,
            default: ''
        },

        channelId : {
            type: String,
            required: true
        }
    },


    components: {
        'altchat-header' : BeesenderChatHeader,
        'altchat-widget-button' : BeesenderChatWidgetButton,
        'altchat-footer' : BeesenderChatFooter,
        'altchat-body' : BeesenderChatBody
    },

    data () {
        return {
            isVisible : false,
            messages: null,
            sendMsgToServerMethod: null,
            sendTypingText: null,
            id: null,
            signalR: signalR
        }
    },

    methods: {
        onSendClick: function (message) {
            if (this.sendMsgToServerMethod) {
                var msg = message.replace(/\n+/g, "\n").trim();
                if (!msg || msg == "") return;
                var clientId = this.clientId;
                this.sendMsgToServerMethod(msg, clientId);
            }
        },
        addMessage: function (msg) {
            if (msg.text != "#[Hello]#" && !msg.text.startsWith('#[UserInfo]#')) {
                if(!this.messages) {
                    this.messages = [];
                }
                this.messages.push(msg);
            }
            this.rerenderFeed();
            var history = {
                time: new Date(),
                messages: this.messages
            };
            var serializiableMessages = JSON.stringify(history);
            localStorage.setItem(this.id, serializiableMessages);
        },

        addOperatorMessage: function (msg) {
            this.addMessage(msg);
        },

        linkify : function(inputText) {
            var replacedText, replacePattern1, replacePattern2, replacePattern3;

            replacePattern1 = /(\b(https?|ftp):\/\/[-A-Z0-9+&@#\/%?=~_|!:,.;]*[-A-Z0-9+&@#\/%=~_|])/gim;
            replacedText = inputText.replace(replacePattern1, '<a href="$1" target="_blank">$1</a>');

            replacePattern2 = /(^|[^\/])(www\.[\S]+(\b|$))/gim;
            replacedText =
                replacedText.replace(replacePattern2, '$1<a href="http://$2" target="_blank">$2</a>');

            replacePattern3 = /(([a-zA-Z0-9\-\_\.])+@[a-zA-Z\_]+?(\.[a-zA-Z]{2,6})+)/gim;
            replacedText = replacedText.replace(replacePattern3, '<a href="mailto:$1">$1</a>');

            return replacedText;
        },

        getClientSimpleMessage: function (text) {
            this.setInputDisabledStatus(false);
            this.clearInputField();
            return {
                text: this.linkify(text),
                classes: "altchat-conversation-part altchat-conversation-part-user",
                classes2: "altchat-comment-container altchat-comment-container-user",
                style: 'background-color: ' + this.themeColor
            }
        },
        getOperatorSimpleMessage: function (text) {
            this.setInputDisabledStatus(false);
            this.clearInputField();
            return {
                text: this.linkify(text),
                classes: "altchat-conversation-part altchat-conversation-part-admin altchat-conversation-part-grouped",
                classes2: "altchat-comment-container altchat-comment-container-admin"
            }
        },
        getOperatorButtonsMessage: function function_name(msg) {
            var message = {
                text: this.linkify(msg.Text),
                classes: "altchat-conversation-part altchat-conversation-part-admin altchat-conversation-part-grouped",
                classes2: "altchat-comment-container altchat-comment-container-admin",
                buttons: []
            };
            for (var i = 0; i < msg.Buttons.length; i++) {
                message.buttons.push({
                    caption: msg.Buttons[i],
                    classes: "altchat-bot-button"
                })
            };
            return message;
        },
        setInputDisabledStatus: function(status) {
            if (this.disableInputOnButtonMessages) {
                var sendBox = document.getElementById('altchat-sendbox-textarea');

                sendBox.setAttribute("disabled", status);
                sendBox.innerHTML = this.chatButtonsPlaceholder;

                [].forEach.call(document.getElementsByClassName('altchat-composer-buttons'), function(element) {
                    element.setAttribute("disabled", status);
                });
            }
        },
        clearInputField: function() {
            document.getElementById('altchat-sendbox-textarea').innerHTML = '';
        },
        getCookie: function(name) {
            var matches = document.cookie.match(new RegExp(
                "(?:^|; )" + name.replace(/([\.$?*|{}\(\)\[\]\\\/\+^])/g, '\\$1') + "=([^;]*)"
            ));
            return matches ? decodeURIComponent(matches[1]) : undefined;
        },
        setCookie: function(name, value, options) {
            options = options || {};

            var expires = options.expires || 100;

            if (typeof expires == "number" && expires) {
                var d = new Date();
                d.setTime(d.getTime() + expires * 10000000);
                expires = options.expires = d;
            }
            if (expires && expires.toUTCString) {
                options.expires = expires.toUTCString();
            }

            value = encodeURIComponent(value);

            var updatedCookie = name + "=" + value;

            for (var propName in options) {
                updatedCookie += "; " + propName;
                var propValue = options[propName];
                if (propValue !== true) {
                    updatedCookie += "=" + propValue;
                }
            }

            document.cookie = updatedCookie;
        },

        unBindClientTypingHandler: function() {
            document.getElementById('altchat-sendbox-textarea')
                .removeEventListener('keyup', typingHandler);
        },

        typingHandler: function(key) {
            if (!key.shiftKey && (key.key === " " || key.key == "Backspace")) {
                this.sendTypingText(document.getElementById('altchat-sendbox-textarea').value);
            }
        },

        rerenderFeed: function() {
            setTimeout(function () {
                var feed = document.getElementById("altchat-main-feed");
                feed.scrollTop = feed.scrollHeight;
            }, 50);
        },

        GetUserBrowserName: function() {
            // Opera 8.0+
            var isOpera = (!!window.opr && !!opr.addons) || !!window.opera || navigator.userAgent.indexOf(' OPR/') >= 0;

            // Firefox 1.0+
            var isFirefox = typeof InstallTrigger !== 'undefined';

            // Safari 3.0+ "[object HTMLElementConstructor]" 
            var isSafari = /constructor/i.test(window.HTMLElement) || (function (p) { return p.toString() === "[object SafariRemoteNotification]"; })(!window['safari'] || (typeof safari !== 'undefined' && safari.pushNotification));

            // Internet Explorer 6-11
            var isIE = /*@cc_on!@*/false || !!document.documentMode;

            // Edge 20+
            var isEdge = !isIE && !!window.StyleMedia;

            // Chrome 1+
            var isChrome = !!window.chrome && !!window.chrome.webstore;

            // Blink engine detection
            var isBlink = (isChrome || isOpera) && !!window.CSS;

            if (isOpera)
                return "Opera";
            if (isFirefox)
                return "Firefox";
            if (isSafari)
                return "Safari";
            if (isIE)
                return "IE";
            if (isEdge)
                return "Edge";
            if (isChrome)
                return "Chrome";
            if (isBlink)
                return "Blink";
        }
    }
}
</script>

<style>
#altchat-container a, #altchat-container abbr, #altchat-container acronym, #altchat-container address, #altchat-container applet, #altchat-container article, #altchat-container aside, #altchat-container audio, #altchat-container b, #altchat-container big, #altchat-container blockquote, #altchat-container button, #altchat-container canvas, #altchat-container caption, #altchat-container center, #altchat-container cite, #altchat-container code, #altchat-container dd, #altchat-container del, #altchat-container details, #altchat-container dfn, #altchat-container div, #altchat-container div.form, #altchat-container dl, #altchat-container dt, #altchat-container em, #altchat-container fieldset, #altchat-container figcaption, #altchat-container figure, #altchat-container footer, #altchat-container form, #altchat-container h1, #altchat-container h2, #altchat-container h3, #altchat-container h4, #altchat-container h5, #altchat-container h6, #altchat-container header, #altchat-container hgroup, #altchat-container i, #altchat-container iframe, #altchat-container img, #altchat-container input, #altchat-container input[type], #altchat-container ins, #altchat-container kbd, #altchat-container label, #altchat-container legend, #altchat-container li, #altchat-container mark, #altchat-container menu, #altchat-container nav, #altchat-container object, #altchat-container ol, #altchat-container p, #altchat-container pre, #altchat-container q, #altchat-container s, #altchat-container samp, #altchat-container section, #altchat-container small, #altchat-container span, #altchat-container strike, #altchat-container strong, #altchat-container sub, #altchat-container summary, #altchat-container sup, #altchat-container table, #altchat-container tbody, #altchat-container td, #altchat-container textarea, #altchat-container tfoot, #altchat-container th, #altchat-container thead, #altchat-container time, #altchat-container tr, #altchat-container tt, #altchat-container u, #altchat-container ul, #altchat-container var, #altchat-container video {
    text-align: left;
    background: none 0 0 repeat scroll padding-box transparent;
    background-color: transparent;
    background-image: none;
    text-decoration: none;
    border: 0 none transparent;
    bottom: auto;
    color: inherit;
    cursor: auto;
    float: none;
    list-style: disc outside none;
    margin: 0;
    -ms-opacity: 1;
    opacity: 1;
    orphans: 2;
    outline: medium none invert;
    outline-offset: 0;
    overflow: visible;
    overflow-style: auto;
    padding: 0;
    right: auto;
}

.altchat-widget-startbutton {
    -moz-filter: drop-shadow(0px 0px 5px rgba(204, 204, 204, 0.68));
    -webkit-filter: drop-shadow(0px 0px 5px rgba(204, 204, 204, 0.68));
    filter: drop-shadow(0px 0px 5px rgba(204, 204, 204, 0.68));
    position: fixed;
    bottom: 20px;
    width: 60px;
    right: 30px;
    height: 60px;
    -ms-background-size: 60px;
    background-size: 60px;
    z-index: 9999;
    -ms-transition: transform 1.5s;
    -o-transition: transform 1.5s;
    -webkit-transition: transform 1.5s;
    transition: transform 1.5s;
}

    .altchat-widget-startbutton img {
        position: absolute;
        width: 60px;
        height: 60px;
    }

.altchat-widget-startbutton-front {
    z-index: 2;
    -webkit-backface-visibility: hidden;
    backface-visibility: hidden;
}

@-webkit-keyframes altchat-widget-startbutton-turn {
    to {
        -webkit-transform: rotateY(360deg);
    }
}


@keyframes turn {
    to {
        -ms-transform: rotateY(360deg);
        -webkit-transform: rotateY(360deg);
        transform: rotateY(360deg);
    }
}

#altchat-container address, #altchat-container article, #altchat-container aside, #altchat-container blockquote, #altchat-container canvas, #altchat-container center, #altchat-container dd, #altchat-container details, #altchat-container dir, #altchat-container div, #altchat-container div.form, #altchat-container dl, #altchat-container dt, #altchat-container fieldset, #altchat-container figcaption, #altchat-container figure, #altchat-container footer, #altchat-container form, #altchat-container frame, #altchat-container frameset, #altchat-container h1, #altchat-container h2, #altchat-container h3, #altchat-container h4, #altchat-container h5, #altchat-container h6, #altchat-container header, #altchat-container hgroup, #altchat-container hr, #altchat-container menu, #altchat-container nav, #altchat-container noframes, #altchat-container ol, #altchat-container p, #altchat-container pre, #altchat-container section, #altchat-container summary, #altchat-container ul {
    display: block
}

#altchat-container a, #altchat-container a *, #altchat-container a span, #altchat-container button, #altchat-container button *, #altchat-container button span, #altchat-container input[type=reset], #altchat-container input[type=submit] {
    cursor: pointer
}

#altchat-container :focus {
    outline: none
}

#altchat-container-body {
    display: block;
    overflow: hidden
}

body > .altchat-container {
    position: fixed
}

#altchat-container .altchat-avatar {
    margin: 0 auto;
    border-radius: 50%;
    display: inline-block;
    vertical-align: middle
}

    #altchat-container .altchat-avatar img {
        border-radius: 50%
    }

#altchat-container .altchat-messenger {
    position: absolute;
    bottom: 0px;
    right: 0px;
}

#altchat-container .altchat-chat {
    z-index: 2147483000;
    position: fixed;
    bottom: 20px;
    right: 20px !important;
    width: 240px;
}

#altchat-container .altchat-header-buttons-close {
    z-index: 2147483000;
    background-position: 50%;
    width: 75px;
    height: 75px
}

    #altchat-container .altchat-header-buttons-close, #altchat-container .altchat-header-buttons-close * {
        cursor: pointer
    }

#altchat-container .altchat-header-buttons-close {
    z-index: 2147483000;
    background-image: url(http://beesender.com/images/close.png);
    background-size: 14px 14px;
    background-repeat: no-repeat;
    position: absolute;
    top: 0;
    right: 0;
    display: none
}

@media (-webkit-min-device-pixel-ratio:1.3), (min--moz-device-pixel-ratio:1.3), (min-device-pixel-ratio:1.3), (min-resolution:1.3dppx) {
    #altchat-container .altchat-header-buttons-close {
        background-image: url(http://beesender.com/images/close2.png)
    }
}

#altchat-container .altchat-header-buttons-close:hover .altchat-header-buttons-close-contents {
    background-color: #000000;
    background-color: rgba(0, 0, 0, .1)
}

#altchat-container .altchat-header-buttons-close.altchat-header-buttons-close-visible {
    display: block
}

#altchat-container .altchat-header-buttons-close-contents {
    color: rgba(0, 0, 0, .3);
    width: 50px;
    text-align: center;
    vertical-align: top;
    height: 50px;
    padding: 15px;
    margin: 13px;
    box-sizing: border-box;
    border-radius: 8px
}

#altchat-container .altchat-conversations {
    position: absolute;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0
}

#altchat-container .altchat-conversations-body {
    position: absolute;
    top: 75px;
    bottom: 0;
    left: 0;
    right: 0;
    overflow-y: auto
}

#altchat-container .altchat-conversations-body-conversations {
    padding-bottom: 105px
}

#altchat-container .altchat-conversations-body-empty-text-container {
    position: relative;
    top: 50%;
    -webkit-transform: translateY(-50%);
    -ms-transform: translateY(-50%);
    transform: translateY(-50%)
}

#altchat-container .altchat-conversations-body-empty-header {
    font-size: 17px;
    text-align: center;
    padding-bottom: 6px
}

#altchat-container .altchat-conversations-body-empty-text {
    font-size: 15px;
    text-align: center;
    color: #8f919d
}

#altchat-container .altchat-conversations-spinner {
    position: absolute;
    left: 50%;
    top: 50%;
    margin-left: -14px;
    margin-top: -14px
}

#altchat-container .altchat-conversations-footer {
    z-index: 2147483001;
    text-align: center;
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    border-radius: 0 0 6px 6px;
    height: 90px;
    pointer-events: none;
    background: linear-gradient(0deg, #fff, hsla(0, 0%, 100%, 0))
}

#altchat-container .altchat-conversation {
    position: absolute;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;
    overflow: hidden
}

#altchat-container .altchat-conversation-footer {
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    border-radius: 0 0 6px 6px
}

#altchat-container .altchat-conversation-body-container {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0
}

#altchat-container .altchat-conversation-body {
    -webkit-transform: translateY(0);
    -ms-transform: translateY(0);
    transform: translateY(0);
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0
}

#altchat-container .altchat-conversation-body-snapped {
    transition: -webkit-transform .15s ease-out;
    transition: transform .15s ease-out;
    transition: transform .15s ease-out, -webkit-transform .15s ease-out
}

#altchat-container .altchat-conversation-body-profile {
    background-color: #1F8CEB;
    position: absolute;
    width: 100%;
    box-shadow: 0 1px 4px rgba(0, 0, 0, .2);
    z-index: 2147483001
}

#altchat-container .altchat-conversation-body-parts-wrapper {
    overflow: hidden
}

#altchat-container .altchat-conversation-body-parts {
    position: absolute;
    left: 0;
    right: 0;
    bottom: 0;
    overflow-y: scroll;
    background-color: white;
}

/* background */


#altchat-container .altchat-conversation-part {
    padding-bottom: 10px;
    animation: fadeIn 2s;
}

    #altchat-container .altchat-conversation-part:after, #altchat-container .altchat-conversation-part:before {
        content: " ";
        display: table;
    }

    #altchat-container .altchat-conversation-part:after {
        clear: both
    }

#altchat-container .altchat-conversation-part-failed * {
    cursor: pointer
}

#altchat-container .altchat-conversation-part-grouped, #altchat-container .altchat-conversation-part-last {
    padding-bottom: 3px
}

#altchat-container .altchat-conversation-part-last, #altchat-container .altchat-conversation-part-last-spaced {
    padding-bottom: 30px
}

#altchat-container .altchat-conversation-part-metadata {
    clear: both;
    color: #b8c3ca;
    font-size: 13px;
    padding-top: 5px;
    width: 75%
}

#altchat-container .altchat-conversation-part-metadata-attribution {
    display: inline
}

#altchat-container .altchat-conversation-part-metadata-save-state {
    white-space: nowrap
}

#altchat-container .altchat-conversation-part-admin .altchat-conversation-part-metadata {
    padding-left: 45px
}

#altchat-container .altchat-conversation-part-user .altchat-conversation-part-metadata, #altchat-container .altchat-conversation-part-user .altchat-conversation-part-metadata-save-state {
    float: right
}

#altchat-container .altchat-conversation-part-other-user .altchat-conversation-part-metadata {
    padding-left: 45px
}

#altchat-container .altchat-conversation-parts {
    padding: 30px 35px 40px
}

#altchat-container .altchat-conversation-parts-date-divider {
    text-align: center;
    color: #b8c3ca;
    font-size: 13px;
    clear: both;
    padding-top: 14px;
    padding-bottom: 14px
}

    #altchat-container .altchat-conversation-parts-date-divider:first-child {
        padding-top: 0
    }

#altchat-container .altchat-team-profile-compact {
    position: absolute;
    height: 75px;
    bottom: 0;
    left: 75px;
    right: 20px;
    padding-top: 12.5px;
    box-sizing: border-box;
    overflow: hidden
}

#altchat-container .altchat-team-profile-compact-avatar-container {
    display: inline-block;
    vertical-align: middle;
    white-space: nowrap;
    float: left;
    padding-right: 10px
}

#altchat-container .altchat-team-profile-compact-avatar {
    position: relative;
    display: inline-block;
    border-radius: 100%
}

    #altchat-container .altchat-team-profile-compact-avatar .altchat-avatar {
        width: 32px;
        height: 32px;
        line-height: 32px;
        font-size: 16px
    }

        #altchat-container .altchat-team-profile-compact-avatar .altchat-avatar img {
            width: 32px;
            height: 32px
        }

#altchat-container .altchat-team-profile-compact-contents {
    border-radius: 8px;
    padding: 8px;
    height: 50px;
    box-sizing: border-box;
    white-space: nowrap;
    overflow: hidden
}

#altchat-container .altchat-team-profile-compact-team-name {
    color: #fff;
    font-size: 17px;
    line-height: 1.1em;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis
}

#altchat-container .altchat-team-profile-compact-response-delay {
    color: hsla(0, 0%, 100%, .8);
    font-size: 13px;
    line-height: 1.3em;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis
}

#altchat-container .altchat-team-profile-response-delay-text {
    white-space: nowrap;
    text-align: center
}

#altchat-container .altchat-team-profile-office-hours .altchat-team-profile-full-avatar-container {
    padding-top: 20px
}

#altchat-container .altchat-team-profile-office-hours .altchat-team-profile-response-delay {
    position: relative;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    font-size: 14px;
    color: #fff
}

    #altchat-container .altchat-team-profile-office-hours .altchat-team-profile-response-delay:before {
        position: absolute;
        display: block;
        top: 0;
        left: 20px;
        right: 20px;
        height: 1px;
        content: " ";
        background: linear-gradient(90deg, transparent 0, hsla(0, 0%, 100%, .1) 25%, hsla(0, 0%, 100%, .1) 75%, transparent)
    }

#altchat-container .altchat-team-profile-office-hours .altchat-team-profile-response-delay-office-hours {
    color: #fff;
    background-color: hsla(0, 0%, 100%, .15)
}

    #altchat-container .altchat-team-profile-office-hours .altchat-team-profile-response-delay-office-hours:before {
        display: none
    }

#altchat-container .altchat-team-profile-office-hours .altchat-team-profile-full {
    padding-bottom: 0
}

#altchat-container .altchat-team-profile-office-hours .altchat-team-profile-full-intro {
    margin-bottom: 30px
}

#altchat-container .altchat-team-profile-office-hours .altchat-team-profile-response-delay-wrapper {
    text-align: center;
    margin: 15px
}

#altchat-container .altchat-team-profile-office-hours .altchat-team-profile-response-delay-help-center-link-wrapper {
    text-align: center;
    margin: 12px
}

#altchat-container .altchat-team-profile-office-hours .altchat-team-profile-response-delay-help-center-link {
    text-decoration: underline
}

    #altchat-container .altchat-team-profile-office-hours .altchat-team-profile-response-delay-help-center-link:hover {
        text-decoration: none
    }

#altchat-container .altchat-admin-profile-compact-contents {
    padding: 8px;
    height: 50px;
    box-sizing: border-box;
    border-radius: 8px
}

#altchat-container .altchat-admin-profile-collapsed, #altchat-container .altchat-admin-profile-collapsed * {
    cursor: pointer
}

    #altchat-container .altchat-admin-profile-collapsed:hover .altchat-admin-profile-compact-contents {
        background-color: rgba(0, 0, 0, .1)
    }

#altchat-container .altchat-admin-profile-compact-body {
    display: block;
    vertical-align: middle;
    padding-left: 10px
}

#altchat-container .altchat-admin-profile-compact-admin-name {
    color: #fff;
    font-size: 16px;
    line-height: 1.2em;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis
}

#altchat-container .altchat-composer {
    position: absolute;
    bottom: 30px !important;
    left: 0;
    right: 0;
    min-height: 55px;
    max-height: 200px;
    border-left: 1px solid #21293d;
}

    #altchat-container .altchat-composer pre, #altchat-container .altchat-composer textarea {
        box-sizing: border-box;
        padding: 18px;
        padding-right: 100px;
        padding-left: 45px;
        width: 100%;
        height: 100%;
        font-size: 15px;
        font-weight: 400;
        line-height: 1.33;
        white-space: pre;
        white-space: pre-wrap;
        word-wrap: break-word
    }

    #altchat-container .altchat-composer textarea {
        position: absolute;
        bottom: 0;
        left: 0;
        color: #565867;
        background-color: #f4f7f9;
        resize: none;
        border: none;
        transition: background-color .2s ease, box-shadow .2s ease;
        overflow: auto;
        height: 60px !important;
    }

        #altchat-container .altchat-composer textarea::-webkit-input-placeholder {
            color: #b8c3ca;
            font-size: 15px;
            font-weight: 400;
            line-height: 1.33;
            opacity: .7
        }

        #altchat-container .altchat-composer textarea::-moz-placeholder {
            color: #b8c3ca;
            font-size: 15px;
            font-weight: 400;
            line-height: 1.33;
            opacity: .7
        }

        #altchat-container .altchat-composer textarea:-ms-input-placeholder {
            color: #b8c3ca;
            font-size: 15px;
            font-weight: 400;
            line-height: 1.33;
            opacity: .7
        }

        #altchat-container .altchat-composer textarea:focus {
            outline: none;
            background-color: #fff;
            box-shadow: 0 0 100px 0 rgba(150, 165, 190, .24)
        }

    #altchat-container .altchat-composer pre {
        visibility: hidden
    }

#altchat-container .altchat-composer-send-button {
    background-size: 20px 20px;
    background-repeat: no-repeat;
    content: " ";
    top: 19px;
    width: 20px;
    height: 20px;
    left: 5px;
}

#altchat-container .altchat-composer-buttons {
    position: absolute;
    top: 0;
    right: 10px
}

#altchat-container .altchat-composer-send-button, #altchat-container .altchat-composer-upload-button {
    position: relative;
    float: left;
    display: inline-block;
    cursor: pointer;
    opacity: .7;
    height: 55px
}

    #altchat-container .altchat-composer-gif-button:hover, #altchat-container .altchat-composer-send-button:hover, #altchat-container .altchat-composer-upload-button:hover {
        opacity: 1
    }

    #altchat-container .altchat-composer-gif-button:before, #altchat-container .altchat-composer-send-button:before, #altchat-container .altchat-composer-upload-button:before {
        position: absolute;
        top: 20px;
        background-position: 50%
    }

#altchat-container .altchat-composer-send-button {
    width: 30px
}

    #altchat-container .altchat-composer-send-button:before {
        background-size: 20px 20px;
        background-repeat: no-repeat;
        content: " ";
        top: 19px;
        width: 20px;
        height: 20px;
        left: 5px
    }

#altchat-container .altchat-composer-upload-button {
    width: 30px
}

    #altchat-container .altchat-composer-upload-button:before {
        background-image: url(http://beesender.com/images/upload-button.png);
        background-size: 16px 18px;
        background-repeat: no-repeat;
        content: " ";
        width: 16px;
        height: 18px;
        left: 5px
    }

@media (-webkit-min-device-pixel-ratio:1.3), (min--moz-device-pixel-ratio:1.3), (min-device-pixel-ratio:1.3), (min-resolution:1.3dppx) {
    #altchat-container .altchat-composer-upload-button:before {
        background-image: url(http://beesender.com/images/upload-button@2x.png)
    }
}

@media only screen and (max-device-width:667px) {
    #altchat-container .altchat-composer-send-button {
        position: relative;
        top: 15px;
        border-radius: 50%;
        width: 36px;
        height: 36px;
        background-size: 20px 20px;
        background-repeat: no-repeat;
        content: " ";
        top: 15px;
        width: 20px;
        height: 20px;
        left: -8px;
    }

        #altchat-container .altchat-composer-send-button:before {
            background-size: 14px 14px;
            background-repeat: no-repeat;
            content: " ";
            position: absolute;
            top: 12px;
            left: 12px;
            width: 14px;
            height: 14px
        }
}

#altchat-container .altchat-comment-container {
    position: relative;
    max-width: 75%
}

#altchat-container .altchat-comment-container-user {
    float: right
}

#altchat-container .altchat-comment-container-admin, #altchat-container .altchat-comment-container-other-user {
    float: left;
    padding-left: 0px
}

#altchat-container .altchat-comment-container-admin-avatar {
    position: absolute;
    left: 0;
    bottom: 10px
}

    #altchat-container .altchat-comment-container-admin-avatar .altchat-avatar {
        width: 30px;
        height: 30px;
        line-height: 30px;
        font-size: 15px
    }

        #altchat-container .altchat-comment-container-admin-avatar .altchat-avatar img {
            width: 30px;
            height: 30px
        }

#altchat-container .altchat-comment-container-admin-borderless-avatar {
    position: absolute;
    bottom: 0;
    left: 0;
    box-shadow: 0 2px 8px 0 rgba(35, 47, 53, .09);
    border-radius: 100%
}

    #altchat-container .altchat-comment-container-admin-borderless-avatar .altchat-avatar {
        width: 30px;
        height: 30px;
        line-height: 30px;
        font-size: 15px
    }

        #altchat-container .altchat-comment-container-admin-borderless-avatar .altchat-avatar img {
            width: 30px;
            height: 30px
        }

#altchat-container .altchat-comment {
    padding: 12px 10px;
    border-radius: 6px;
    position: relative;
    overflow-wrap: break-word;
    white-space: pre-line;
}

    #altchat-container .altchat-comment pre span {
        color: inherit !important;
        background-color: inherit !important;
        font-weight: inherit !important;
        word-wrap: break-word
    }

#altchat-container .altchat-comment-single .altchat-image-progress {
    border-radius: 6px
}

#altchat-container .altchat-conversation-part-failed .altchat-comment {
    opacity: .8
}

#altchat-container .altchat-comment-container-user .altchat-comment {
    color: #fff;
    background-color: rgb(57, 153, 237) !important;
}

#altchat-container .altchat-comment-container-admin .altchat-comment {
    color: #3898ed !important;
    background-color: #f4f7f9
}

    #altchat-container .altchat-comment-container-admin .altchat-comment a {
        color: #263238;
        text-decoration: underline
    }

    #altchat-container .altchat-comment-container-admin .altchat-comment .altchat-block-button-container {
        margin-bottom: 10px
    }

    #altchat-container .altchat-comment-container-admin .altchat-comment .altchat-block-button:hover {
        text-decoration: none
    }

#altchat-container .altchat-comment-container-other-user .altchat-comment {
    color: #263238;
    background-color: #f4f7f9
}

    #altchat-container .altchat-comment-container-other-user .altchat-comment a {
        color: #263238;
        text-decoration: underline
    }

    #altchat-container .altchat-comment-container-other-user .altchat-comment .altchat-block-button-container {
        margin-bottom: 10px
    }

    #altchat-container .altchat-comment-container-other-user .altchat-comment .altchat-block-button:hover {
        text-decoration: none
    }

#altchat-container .altchat-comment .altchat-block-paragraph {
    font-size: 14px;
    line-height: 1.4;
    margin: 0 0 10px;
    font-family: Arial;
}

@-webkit-keyframes altchat-messenger-view-fade-in {
    0% {
        opacity: 0
    }

    to {
        opacity: 1
    }
}

@keyframes altchat-messenger-view-fade-in {
    0% {
        opacity: 0
    }

    to {
        opacity: 1
    }
}

@-webkit-keyframes altchat-messenger-view-fade-out {
    0% {
        opacity: 1
    }

    to {
        opacity: 0
    }
}

@keyframes altchat-messenger-view-fade-out {
    0% {
        opacity: 1
    }

    to {
        opacity: 0
    }
}

@-webkit-keyframes altchat-composer-slide-in {
    0% {
        opacity: 0;
        -webkit-transform: translateY(15px);
        transform: translateY(15px)
    }

    to {
        opacity: 1;
        -webkit-transform: translateY(0);
        transform: translateY(0)
    }
}

@keyframes altchat-composer-slide-in {
    0% {
        opacity: 0;
        -webkit-transform: translateY(15px);
        transform: translateY(15px)
    }

    to {
        opacity: 1;
        -webkit-transform: translateY(0);
        transform: translateY(0)
    }
}

@-webkit-keyframes altchat-composer-slide-out {
    0% {
        opacity: 1;
        -webkit-transform: translateY(0);
        transform: translateY(0)
    }

    to {
        opacity: 0;
        -webkit-transform: translateY(15px);
        transform: translateY(15px)
    }
}

@keyframes altchat-composer-slide-out {
    0% {
        opacity: 1;
        -webkit-transform: translateY(0);
        transform: translateY(0)
    }

    to {
        opacity: 0;
        -webkit-transform: translateY(15px);
        transform: translateY(15px)
    }
}

@-webkit-keyframes altchat-messenger-view-slide-left-out {
    0% {
        opacity: 1;
        -webkit-transform: translateX(0);
        transform: translateX(0)
    }

    to {
        opacity: 0;
        -webkit-transform: translateX(-8px);
        transform: translateX(-8px)
    }
}

@keyframes altchat-messenger-view-slide-left-out {
    0% {
        opacity: 1;
        -webkit-transform: translateX(0);
        transform: translateX(0)
    }

    to {
        opacity: 0;
        -webkit-transform: translateX(-8px);
        transform: translateX(-8px)
    }
}

@-webkit-keyframes altchat-messenger-view-slide-right-out {
    0% {
        opacity: 1;
        -webkit-transform: translateX(0);
        transform: translateX(0)
    }

    to {
        opacity: 0;
        -webkit-transform: translateX(8px);
        transform: translateX(8px)
    }
}

@keyframes altchat-messenger-view-slide-right-out {
    0% {
        opacity: 1;
        -webkit-transform: translateX(0);
        transform: translateX(0)
    }

    to {
        opacity: 0;
        -webkit-transform: translateX(8px);
        transform: translateX(8px)
    }
}

@-webkit-keyframes altchat-messenger-view-slide-left-in {
    0% {
        opacity: 0;
        -webkit-transform: translateX(8px);
        transform: translateX(8px)
    }

    to {
        opacity: 1;
        -webkit-transform: translateX(0);
        transform: translateX(0)
    }
}

@keyframes altchat-messenger-view-slide-left-in {
    0% {
        opacity: 0;
        -webkit-transform: translateX(8px);
        transform: translateX(8px)
    }

    to {
        opacity: 1;
        -webkit-transform: translateX(0);
        transform: translateX(0)
    }
}

@-webkit-keyframes altchat-messenger-view-slide-right-in {
    0% {
        opacity: 0;
        -webkit-transform: translateX(-8px);
        transform: translateX(-8px)
    }

    to {
        opacity: 1;
        -webkit-transform: translateX(0);
        transform: translateX(0)
    }
}

@keyframes altchat-messenger-view-slide-right-in {
    0% {
        opacity: 0;
        -webkit-transform: translateX(-8px);
        transform: translateX(-8px)
    }

    to {
        opacity: 1;
        -webkit-transform: translateX(0);
        transform: translateX(0)
    }
}

@-webkit-keyframes mv3-new-conversation-button-slide-down {
    0% {
        opacity: 1;
        -webkit-transform: translateY(0) translateX(-50%) scale(1);
        transform: translateY(0) translateX(-50%) scale(1)
    }

    to {
        opacity: 0;
        -webkit-transform: translateY(8px) translateX(-50%) scale(.96);
        transform: translateY(8px) translateX(-50%) scale(.96)
    }
}

@keyframes mv3-new-conversation-button-slide-down {
    0% {
        opacity: 1;
        -webkit-transform: translateY(0) translateX(-50%) scale(1);
        transform: translateY(0) translateX(-50%) scale(1)
    }

    to {
        opacity: 0;
        -webkit-transform: translateY(8px) translateX(-50%) scale(.96);
        transform: translateY(8px) translateX(-50%) scale(.96)
    }
}

@-webkit-keyframes mv3-new-conversation-button-slide-up {
    0% {
        opacity: 0;
        -webkit-transform: translateY(8px) translateX(-50%) scale(.96);
        transform: translateY(8px) translateX(-50%) scale(.96)
    }

    to {
        opacity: 1;
        -webkit-transform: translateY(0) translateX(-50%) scale(1);
        transform: translateY(0) translateX(-50%) scale(1)
    }
}

@keyframes mv3-new-conversation-button-slide-up {
    0% {
        opacity: 0;
        -webkit-transform: translateY(8px) translateX(-50%) scale(.96);
        transform: translateY(8px) translateX(-50%) scale(.96)
    }

    to {
        opacity: 1;
        -webkit-transform: translateY(0) translateX(-50%) scale(1);
        transform: translateY(0) translateX(-50%) scale(1)
    }
}

#altchat-container .altchat-messenger-view-enter.altchat-conversations {
    z-index: 2147483001
}

    #altchat-container .altchat-messenger-view-enter.altchat-conversations .altchat-conversations-new-conversation-button {
        -webkit-animation-name: mv3-new-conversation-button-slide-up;
        animation-name: mv3-new-conversation-button-slide-up;
        -webkit-animation-duration: .32s;
        animation-duration: .32s;
        -webkit-animation-timing-function: cubic-bezier(.23, 1, .32, 1);
        animation-timing-function: cubic-bezier(.23, 1, .32, 1);
        -webkit-animation-delay: .16s;
        animation-delay: .16s;
        -webkit-animation-fill-mode: both;
        animation-fill-mode: both
    }

#altchat-container .altchat-conversations {
    -webkit-animation-name: altchat-messenger-view-fade-in;
    animation-name: altchat-messenger-view-fade-in;
    -webkit-animation-duration: .32s;
    animation-duration: .32s;
    -webkit-animation-timing-function: cubic-bezier(.23, 1, .32, 1);
    animation-timing-function: cubic-bezier(.23, 1, .32, 1);
    -webkit-animation-delay: .16s;
    animation-delay: .16s;
    -webkit-animation-fill-mode: both;
    animation-fill-mode: both
}

#altchat-container .altchat-conversations {
    -webkit-animation-name: altchat-messenger-view-slide-right-in;
    animation-name: altchat-messenger-view-slide-right-in;
    -webkit-animation-duration: .32s;
    animation-duration: .32s;
    -webkit-animation-timing-function: cubic-bezier(.23, 1, .32, 1);
    animation-timing-function: cubic-bezier(.23, 1, .32, 1);
    -webkit-animation-delay: .16s;
    animation-delay: .16s;
    -webkit-animation-fill-mode: both;
    animation-fill-mode: both
}

#altchat-container .altchat-conversations {
    z-index: 2147483000
}


#altchat-container .altchat-conversation .altchat-conversation-backgrounds, #altchat-container .altchat-messenger-view-leave.altchat-conversation .altchat-header-buttons, #altchat-container .altchat-messenger-view-leave.altchat-conversation .altchat-link {
    -webkit-animation-name: altchat-messenger-view-fade-out;
    animation-name: altchat-messenger-view-fade-out;
    -webkit-animation-duration: .32s;
    animation-duration: .32s;
    -webkit-animation-timing-function: cubic-bezier(.23, 1, .32, 1);
    animation-timing-function: cubic-bezier(.23, 1, .32, 1);
    -webkit-animation-delay: .12s;
    animation-delay: .12s;
    -webkit-animation-fill-mode: both;
    animation-fill-mode: both
}

#altchat-container .altchat-messenger-view-leave.altchat-conversation .altchat-conversation-parts {
    -webkit-animation-name: altchat-messenger-view-slide-right-out;
    animation-name: altchat-messenger-view-slide-right-out;
    -webkit-animation-duration: .32s;
    animation-duration: .32s;
    -webkit-animation-timing-function: cubic-bezier(.23, 1, .32, 1);
    animation-timing-function: cubic-bezier(.23, 1, .32, 1);
    -webkit-animation-delay: 20ms;
    animation-delay: 20ms;
    -webkit-animation-fill-mode: both;
    animation-fill-mode: both
}

#altchat-container .altchat-messenger-view-leave.altchat-conversation .altchat-conversation-footer {
    -webkit-animation-name: altchat-composer-slide-out;
    animation-name: altchat-composer-slide-out;
    -webkit-animation-duration: .32s;
    animation-duration: .32s;
    -webkit-animation-timing-function: cubic-bezier(.23, 1, .32, 1);
    animation-timing-function: cubic-bezier(.23, 1, .32, 1);
    -webkit-animation-delay: .12s;
    animation-delay: .12s;
    -webkit-animation-fill-mode: both;
    animation-fill-mode: both
}

#altchat-container .altchat-messenger-view-leave-active.altchat-conversation .altchat-conversation-profile {
    opacity: 0;
    transition-property: opacity, max-height;
    transition-duration: .32s;
    transition-timing-function: cubic-bezier(.23, 1, .32, 1);
    transition-delay: 0
}

#altchat-container .altchat-messenger-view-leave-active.altchat-conversation .altchat-conversation-profile-expanded {
    max-height: 75px
}

#altchat-container .altchat-conversation .altchat-conversation-parts {
    opacity: 1;
    -webkit-transform: translateX(0);
    -ms-transform: translateX(0);
    transform: translateX(0);
    -webkit-animation-name: altchat-messenger-view-slide-left-in;
    animation-name: altchat-messenger-view-slide-left-in;
    -webkit-animation-duration: .32s;
    animation-duration: .32s;
    -webkit-animation-timing-function: cubic-bezier(.23, 1, .32, 1);
    animation-timing-function: cubic-bezier(.23, 1, .32, 1);
    -webkit-animation-delay: .23s;
    animation-delay: .23s;
    -webkit-animation-fill-mode: backwards;
    animation-fill-mode: backwards
}

#altchat-container .altchat-conversation .altchat-conversation-footer {
    opacity: 1;
    -webkit-transform: translateY(0);
    -ms-transform: translateY(0);
    transform: translateY(0);
    -webkit-animation-name: altchat-composer-slide-in;
    animation-name: altchat-composer-slide-in;
    -webkit-animation-duration: .32s;
    animation-duration: .32s;
    -webkit-animation-timing-function: cubic-bezier(.23, 1, .32, 1);
    animation-timing-function: cubic-bezier(.23, 1, .32, 1);
    -webkit-animation-delay: 80ms;
    animation-delay: 80ms;
    -webkit-animation-fill-mode: both;
    animation-fill-mode: both;
    background-color: #21293d;
}

#altchat-container .altchat-conversation.altchat-conversation-fetching .altchat-conversation-backgrounds, #altchat-container .altchat-conversation.altchat-conversation-fetching .altchat-conversation-footer, #altchat-container .altchat-conversation.altchat-conversation-fetching .altchat-conversation-parts, #altchat-container .altchat-conversation.altchat-conversation-fetching .altchat-link {
    opacity: 0;
    -webkit-animation: none;
    animation: none;
    background-color: white;
}

#altchat-container .altchat-conversation-part-user.altchat-conversation-part-enter {
    opacity: 0;
    -webkit-transform: translateY(40px);
    -ms-transform: translateY(40px);
    transform: translateY(40px)
}

    #altchat-container .altchat-conversation-part-user.altchat-conversation-part-enter.altchat-conversation-part-enter-active {
        opacity: 1;
        -webkit-transform: translateY(0);
        -ms-transform: translateY(0);
        transform: translateY(0);
        transition: opacity .2s, -webkit-transform .2s;
        transition: opacity .2s, transform .2s;
        transition: opacity .2s, transform .2s, -webkit-transform .2s
    }

#altchat-container .altchat-conversation-parts-scrolled .altchat-conversation-part-user.altchat-conversation-part-enter {
    -webkit-transform: translateY(0);
    -ms-transform: translateY(0);
    transform: translateY(0)
}

#altchat-container .altchat-conversation-part-admin.altchat-conversation-part-enter {
    opacity: 0
}

    #altchat-container .altchat-conversation-part-admin.altchat-conversation-part-enter.altchat-conversation-part-enter-active {
        opacity: 1;
        transition: opacity .2s
    }

#altchat-container .altchat-conversation-parts {
    transition: -webkit-transform .2s;
    transition: transform .2s;
    transition: transform .2s, -webkit-transform .2s
}

#altchat-container .altchat-conversation-parts-scrolling {
    transition: none
}

@media only screen and (max-device-width:667px), screen and (max-width:369px), screen and (min-height:591px), screen and (min-width:371px) {
    .altchat-messenger {
        -webkit-text-size-adjust: 100%
    }

    .altchat-admin-profile-collapsed:hover .altchat-admin-profile-compact-contents, .altchat-admin-profile-collapsed:hover .altchat-team-profile-compact-contents, .altchat-team-profile-collapsed:hover .altchat-admin-profile-compact-contents, .altchat-team-profile-collapsed:hover .altchat-team-profile-compact-contents {
        background-color: inherit
    }

    .altchat-sticker-native {
        font-size: inherit
    }

    .altchat-composer textarea {
        background-color: #f4f7f9
    }

    .altchat-header-buttons-close {
        display: block !important
    }
}

textarea {
    box-sizing: border-box;
    resize: none;
}


.altchat-bot-buttons {
    margin: 10px auto !important;
}

.altchat-bot-button {
    text-align: center !important;
    border: 1px solid #1f8ceb !important;
    width: 90% !important;
    border-radius: 15px !important;
    padding: 5px !important;
    margin: 5px auto !important;
    color: #1f8ceb !important;
    font-size: 14px !important;
}

.altchat-footer-description {
    z-index: 11 !important;
    height: 20px !important;
    position: absolute !important;
    bottom: 0 !important;
    color: #d0d1d0 !important;
    right: 10px !important;
    padding: 5px auto !important;
    font-size: 12px !important;
}

.altchat-composer-upload-button {
    display: none;
}

#altchat-main-feed {
    bottom: 55px;
    top: 302px;
    border-left: 1px solid #21293d !important;
    background-color: #2b3246c4 !important;
}

@media only screen and (max-device-width: 667px) {

    div.altchat-messenger {
        width: 100% !important;
        height: 100% !important;
        position: fixed;
        left: 0px;
        top: 0px;
        right: 0px;
    }
}

@media screen and (max-width: 980px) {
    #altchat-container .altchat-comment .altchat-block-paragraph {
        font-size: 17px !important;
    }

    #altchat-container .altchat-composer pre, #altchat-container .altchat-composer textarea {
        font-size: 18px !important;
    }


    #altchat-container .altchat-team-profile-compact-team-name {
        font-size: 21px !important;
    }

    .altchat-bot-button {
        font-size: 17px !important;
    }

    .altchat-footer-description {
        font-size: 15px !important;
    }
}

.altchat-messenger {
    z-index: 999 !important;
}






























.beesender-widget-button-shadow {
    position: fixed;
    background: rgba(33, 33, 33, .3);
    width: 100%;
    height: 100%;
    top: 0;
    left: 0;
    visibility: hidden;
    z-index: 10100
}


.beesender-widget-button-inner-container {
    position: relative;
    display: inline-block
}

.beesender-widget-button-position-fixed {
    position: fixed;
    z-index: 10000
}

.beesender-widget-button-block {
    width: 66px;
    height: 66px;
    border-radius: 100%;
    box-sizing: border-box;
    overflow: hidden;
    cursor: pointer
}

    .beesender-widget-button-block .beesender-widget-button-icon {
        opacity: 1
    }

.beesender-widget-button-block-active .beesender-widget-button-icon {
    opacity: .7
}

.beesender-widget-button-position-top-left {
    top: 50px;
    left: 50px
}

.beesender-widget-button-position-top-middle {
    top: 50px;
    left: 50%;
    margin: 0 0 0 -33px
}

.beesender-widget-button-position-top-right {
    top: 50px;
    right: 50px
}

.beesender-widget-button-position-bottom-left {
    left: 50px;
    bottom: 50px
}

.beesender-widget-button-position-bottom-middle {
    left: 50%;
    bottom: 50px;
    margin: 0 0 0 -33px
}

.beesender-widget-button-position-bottom-right {
    right: 50px;
    bottom: 50px
}

.beesender-widget-button-inner-block {
    position: relative;
    display: -webkit-box;
    display: -ms-flexbox;
    display: flex;
    height: 66px;
    border-radius: 100px;
    background: #00aeef;
    box-sizing: border-box
}

.beesender-widget-button-icon-container {
    position: relative
}

.beesender-widget-button-inner-item {
    position: absolute;
    top: 0;
    left: 0;
    padding: 16px 13px;
    -webkit-transition: opacity .6s ease-out;
    transition: opacity .6s ease-out;
    -webkit-animation: socialRotateBack .4s;
    animation: socialRotateBack .4s;
    opacity: 0
}

.beesender-widget-button-icon-animation {
    opacity: 1
}

.beesender-widget-button-inner-mask {
    background: rgb(33, 41, 61) !important;
    position: absolute;
    top: -8px;
    left: -8px;
    height: 82px;
    min-width: 66px;
    -webkit-width: calc(100% + 16px);
    width: calc(100% + 16px);
    border-radius: 100px;
    opacity: .2
}

.beesender-widget-button-icon {
    -webkit-transition: opacity .3s ease-out;
    transition: opacity .3s ease-out;
    cursor: pointer
}

    .beesender-widget-button-icon:hover {
        opacity: 1
    }

.beesender-widget-button-inner-item-active .beesender-widget-button-icon {
    opacity: 1
}

.beesender-widget-button-wrapper {
    position: fixed;
    display: -webkit-box;
    display: -ms-flexbox;
    display: flex;
    -webkit-box-orient: vertical;
    -webkit-box-direction: normal;
    -ms-flex-direction: column;
    flex-direction: column;
    -webkit-box-align: end;
    -ms-flex-align: end;
    align-items: flex-end;
    visibility: hidden;
    z-index: 10150
}

.bx-imopenlines-config-sidebar {
    z-index: 10101
}

.beesender-widget-button-visible {
    visibility: visible;
    -webkit-animation: beesender-widget-button-visible 1s ease-out forwards 1;
    animation: beesender-widget-button-visible 1s ease-out forwards 1
}

@-webkit-keyframes beesender-widget-button-visible {
    0% {
        -webkit-transform: scale(0);
        transform: scale(0)
    }

    30.001% {
        -webkit-transform: scale(1.2);
        transform: scale(1.2)
    }

    62.999% {
        -webkit-transform: scale(1);
        transform: scale(1)
    }

    100% {
        -webkit-transform: scale(1);
        transform: scale(1)
    }
}

@keyframes beesender-widget-button-visible {
    0% {
        -webkit-transform: scale(0);
        transform: scale(0)
    }

    30.001% {
        -webkit-transform: scale(1.2);
        transform: scale(1.2)
    }

    62.999% {
        -webkit-transform: scale(1);
        transform: scale(1)
    }

    100% {
        -webkit-transform: scale(1);
        transform: scale(1)
    }
}

.beesender-widget-button-disable {
    -webkit-animation: beesender-widget-button-disable .3s ease-out forwards 1;
    animation: beesender-widget-button-disable .3s ease-out forwards 1
}

@-webkit-keyframes beesender-widget-button-disable {
    0% {
        -webkit-transform: scale(1);
        transform: scale(1)
    }

    50.001% {
        -webkit-transform: scale(.5);
        transform: scale(.5)
    }

    92.999% {
        -webkit-transform: scale(0);
        transform: scale(0)
    }

    100% {
        -webkit-transform: scale(0);
        transform: scale(0)
    }
}

@keyframes beesender-widget-button-disable {
    0% {
        -webkit-transform: scale(1);
        transform: scale(1)
    }

    50.001% {
        -webkit-transform: scale(.5);
        transform: scale(.5)
    }

    92.999% {
        -webkit-transform: scale(0);
        transform: scale(0)
    }

    100% {
        -webkit-transform: scale(0);
        transform: scale(0)
    }
}

.beesender-widget-button-social {
    display: none
}

.beesender-widget-button-social-item {
    position: relative;
    display: block;
    margin: 0 10px 10px 0;
    width: 45px;
    height: 44px;
    background-color: #fff;
    background-size: 100%;
    border-radius: 25px;
    -webkit-box-shadow: 0 8px 6px -6px rgba(33, 33, 33, .2);
    -moz-box-shadow: 0 8px 6px -6px rgba(33, 33, 33, .2);
    box-shadow: 0 8px 6px -6px rgba(33, 33, 33, .2);
    cursor: pointer
}

    .beesender-widget-button-social-item:hover {
        -webkit-box-shadow: 0 0 6px rgba(0, 0, 0, .16), 0 6px 12px rgba(0, 0, 0, .32);
        box-shadow: 0 0 6px rgba(0, 0, 0, .16), 0 6px 12px rgba(0, 0, 0, .32);
        -webkit-transition: box-shadow .17s cubic-bezier(0, 0, .2, 1);
        transition: box-shadow .17s cubic-bezier(0, 0, .2, 1)
    }

.connector-icon-45 {
    width: 45px;
    height: 45px
}

.beesender-widget-button-callback {
    background-image: url('data:image/svg+xml;charset=US-ASCII,%3Csvg%20xmlns%3D%22http%3A//www.w3.org/2000/svg%22%20width%3D%2229%22%20height%3D%2230%22%20viewBox%3D%220%200%2029%2030%22%3E%3Cpath%20fill%3D%22%23FFF%22%20fill-rule%3D%22evenodd%22%20d%3D%22M21.872%2019.905c-.947-.968-2.13-.968-3.072%200-.718.737-1.256.974-1.962%201.723-.193.206-.356.25-.59.112-.466-.262-.96-.474-1.408-.76-2.082-1.356-3.827-3.098-5.372-5.058-.767-.974-1.45-2.017-1.926-3.19-.096-.238-.078-.394.11-.587.717-.718.96-.98%201.665-1.717.984-1.024.984-2.223-.006-3.253-.56-.586-1.103-1.397-1.56-2.034-.458-.636-.817-1.392-1.403-1.985C5.4%202.2%204.217%202.2%203.275%203.16%202.55%203.9%201.855%204.654%201.12%205.378.438%206.045.093%206.863.02%207.817c-.114%201.556.255%203.023.774%204.453%201.062%202.96%202.68%205.587%204.642%207.997%202.65%203.26%205.813%205.837%209.513%207.698%201.665.836%203.39%201.48%205.268%201.585%201.292.075%202.415-.262%203.314-1.304.616-.712%201.31-1.36%201.962-2.042.966-1.01.972-2.235.012-3.234-1.147-1.192-2.48-1.88-3.634-3.065zm-.49-5.36l.268-.047c.583-.103.953-.707.79-1.295-.465-1.676-1.332-3.193-2.537-4.445-1.288-1.33-2.857-2.254-4.59-2.708-.574-.15-1.148.248-1.23.855l-.038.28c-.07.522.253%201.01.747%201.142%201.326.355%202.53%201.064%203.517%202.086.926.958%201.59%202.125%201.952%203.412.14.5.624.807%201.12.72zm2.56-9.85C21.618%202.292%2018.74.69%2015.56.02c-.56-.117-1.1.283-1.178.868l-.038.28c-.073.537.272%201.04.786%201.15%202.74.584%205.218%201.968%207.217%204.03%201.885%201.95%203.19%204.36%203.803%207.012.122.53.617.873%201.136.78l.265-.046c.57-.1.934-.678.8-1.26-.71-3.08-2.223-5.873-4.41-8.14z%22/%3E%3C/svg%3E');
    background-repeat: no-repeat;
    background-position: center;
    background-color: #00aeef;
    background-size: 43%
}

.beesender-widget-button-crmform {
    background-image: url('data:image/svg+xml;charset=US-ASCII,%3Csvg%20xmlns%3D%22http%3A//www.w3.org/2000/svg%22%20width%3D%2224%22%20height%3D%2224%22%20viewBox%3D%220%200%2024%2024%22%3E%3Cpath%20fill%3D%22%23FFF%22%20fill-rule%3D%22evenodd%22%20d%3D%22M22.407%200h-21.1C.586%200%200%20.586%200%201.306v21.1c0%20.72.586%201.306%201.306%201.306h21.1c.72%200%201.306-.586%201.306-1.305V1.297C23.702.587%2023.117%200%2022.407%200zm-9.094%2018.046c0%20.41-.338.737-.738.737H3.9c-.41%200-.738-.337-.738-.737v-1.634c0-.408.337-.737.737-.737h8.675c.41%200%20.738.337.738.737v1.634zm7.246-5.79c0%20.408-.338.737-.738.737H3.89c-.41%200-.737-.338-.737-.737v-1.634c0-.41.337-.737.737-.737h15.923c.41%200%20.738.337.738.737v1.634h.01zm0-5.8c0%20.41-.338.738-.738.738H3.89c-.41%200-.737-.338-.737-.738V4.822c0-.408.337-.737.737-.737h15.923c.41%200%20.738.338.738.737v1.634h.01z%22/%3E%3C/svg%3E');
    background-repeat: no-repeat;
    background-position: center;
    background-color: #00aeef;
    background-size: 43%
}

.beesender-widget-button-openline_livechat {
    background-image: url('data:image/svg+xml;charset=US-ASCII,%3Csvg%20xmlns%3D%22http%3A//www.w3.org/2000/svg%22%20width%3D%2231%22%20height%3D%2228%22%20viewBox%3D%220%200%2031%2028%22%3E%3Cpath%20fill%3D%22%23FFF%22%20fill-rule%3D%22evenodd%22%20d%3D%22M23.29%2013.25V2.84c0-1.378-1.386-2.84-2.795-2.84h-17.7C1.385%200%200%201.462%200%202.84v10.41c0%201.674%201.385%203.136%202.795%202.84H5.59v5.68h.93c.04%200%20.29-1.05.933-.947l3.726-4.732h9.315c1.41.296%202.795-1.166%202.795-2.84zm2.795-3.785v4.733c.348%202.407-1.756%204.558-4.658%204.732h-8.385l-1.863%201.893c.22%201.123%201.342%202.127%202.794%201.893h7.453l2.795%203.786c.623-.102.93.947.93.947h.933v-4.734h1.863c1.57.234%202.795-1.02%202.795-2.84v-7.57c0-1.588-1.225-2.84-2.795-2.84h-1.863z%22/%3E%3C/svg%3E');
    background-repeat: no-repeat;
    background-position: center;
    background-color: #00aeef;
    background-size: 43%
}

.beesender-widget-button-social-tooltip {
    position: absolute;
    top: 50%;
    left: -9000px;
    display: inline-block;
    padding: 5px 10px;
    border-radius: 10px;
    font: 13px/15px "Helvetica Neue", Arial, Helvetica, sans-serif;
    color: #000;
    background: #fff;
    text-align: center;
    -webkit-transform: translate(0, -50%);
    transform: translate(0, -50%);
    -webkit-transition: opacity .6s linear;
    transition: opacity .6s linear;
    opacity: 0
}

.beesender-widget-button-social-item:hover .beesender-widget-button-social-tooltip {
    left: 50px;
    -webkit-transform: translate(0%, -50%);
    transform: translate(0%, -50%);
    opacity: 1;
    z-index: 1
}

.beesender-widget-button-close {
    display: none
}

.beesender-widget-button-position-bottom-left .beesender-widget-button-social-item:hover .beesender-widget-button-social-tooltip, .beesender-widget-button-position-top-left .beesender-widget-button-social-item:hover .beesender-widget-button-social-tooltip {
    left: 50px;
    -webkit-transform: translate(0%, -50%);
    transform: translate(0%, -50%);
    opacity: 1
}

.beesender-widget-button-position-top-right .beesender-widget-button-social-item:hover .beesender-widget-button-social-tooltip, .beesender-widget-button-position-bottom-right .beesender-widget-button-social-item:hover .beesender-widget-button-social-tooltip {
    left: -5px;
    -webkit-transform: translate(-100%, -50%);
    transform: translate(-100%, -50%);
    opacity: 1
}

.beesender-widget-button-inner-container, .bx-touch .beesender-widget-button-inner-container {
    -webkit-transform: scale(.85);
    transform: scale(.85);
    -webkit-transition: transform .3s;
    transition: transform .3s
}

.beesender-widget-button-top .beesender-widget-button-inner-container, .beesender-widget-button-bottom .beesender-widget-button-inner-container {
    -webkit-transform: scale(.7);
    transform: scale(.7);
    -webkit-transition: transform .3s linear;
    transition: transform .3s linear
}

.beesender-widget-button-top .beesender-widget-button-inner-block, .beesender-widget-button-top .beesender-widget-button-inner-mask, .beesender-widget-button-bottom .beesender-widget-button-inner-block, .beesender-widget-button-bottom .beesender-widget-button-inner-mask {
    background: #d6d6d6 !important;
    -webkit-transition: background .3s linear;
    transition: background .3s linear
}

.beesender-widget-button-top .beesender-widget-button-pulse, .beesender-widget-button-bottom .beesender-widget-button-pulse {
    display: none
}

.beesender-widget-button-wrapper.beesender-widget-button-position-bottom-right, .beesender-widget-button-wrapper.beesender-widget-button-position-bottom-middle, .beesender-widget-button-wrapper.beesender-widget-button-position-bottom-left {
    -webkit-box-orient: vertical;
    -webkit-box-direction: reverse;
    -ms-flex-direction: column-reverse;
    flex-direction: column-reverse
}

.beesender-widget-button-bottom .beesender-widget-button-social, .beesender-widget-button-top .beesender-widget-button-social {
    display: -webkit-box;
    display: -ms-flexbox;
    display: flex;
    -webkit-box-orient: vertical;
    -webkit-box-direction: reverse;
    -ms-flex-direction: column-reverse;
    flex-direction: column-reverse;
    -ms-flex-wrap: wrap;
    flex-wrap: wrap;
    -ms-flex-line-pack: end;
    align-content: flex-end;
    height: -webkit-calc(100vh - 110px);
    height: calc(100vh - 110px);
    -webkit-animation: bottomOpen .3s;
    animation: bottomOpen .3s;
    visibility: visible
}

.beesender-widget-button-top .beesender-widget-button-social {
    -webkit-box-orient: vertical;
    -webkit-box-direction: normal;
    -ms-flex-direction: column;
    flex-direction: column;
    -ms-flex-wrap: wrap;
    flex-wrap: wrap;
    -webkit-box-align: start;
    -ms-flex-align: start;
    align-items: flex-start;
    padding: 10px 0 0 0;
    -webkit-animation: topOpen .3s;
    animation: topOpen .3s
}

.beesender-widget-button-position-bottom-left.beesender-widget-button-bottom .beesender-widget-button-social {
    -ms-flex-line-pack: start;
    align-content: flex-start
}

.beesender-widget-button-position-top-left.beesender-widget-button-top .beesender-widget-button-social {
    -ms-flex-line-pack: start;
    align-content: flex-start
}

.beesender-widget-button-position-top-right.beesender-widget-button-top .beesender-widget-button-social {
    -ms-flex-line-pack: start;
    align-content: flex-start;
    -ms-flex-wrap: wrap-reverse;
    flex-wrap: wrap-reverse
}

.beesender-widget-button-position-bottom-right.beesender-widget-button-bottom .beesender-widget-button-social, .beesender-widget-button-position-bottom-left.beesender-widget-button-bottom .beesender-widget-button-social, .beesender-widget-button-position-bottom-middle.beesender-widget-button-bottom .beesender-widget-button-social {
    -ms-flex-line-pack: start;
    align-content: flex-start;
    -ms-flex-wrap: wrap-reverse;
    flex-wrap: wrap-reverse;
    order: 1
}

.beesender-widget-button-position-bottom-left.beesender-widget-button-bottom .beesender-widget-button-social {
    -ms-flex-wrap: wrap;
    flex-wrap: wrap
}

.beesender-widget-button-position-bottom-left .beesender-widget-button-social-item, .beesender-widget-button-position-top-left .beesender-widget-button-social-item, .beesender-widget-button-position-top-middle .beesender-widget-button-social-item, .beesender-widget-button-position-bottom-middle .beesender-widget-button-social-item {
    margin: 0 0 10px 10px
}

.beesender-widget-button-position-bottom-left.beesender-widget-button-wrapper {
    -webkit-box-align: start;
    -ms-flex-align: start;
    align-items: flex-start
}

.beesender-widget-button-position-top-left.beesender-widget-button-wrapper {
    -webkit-box-align: start;
    -ms-flex-align: start;
    align-items: flex-start
}

.beesender-widget-button-position-bottom-middle.beesender-widget-button-wrapper, .beesender-widget-button-position-top-middle.beesender-widget-button-wrapper {
    -webkit-box-align: start;
    -ms-flex-align: start;
    align-items: flex-start;
    -ms-flex-line-pack: start;
    align-content: flex-start
}

.beesender-widget-button-position-top-middle.beesender-widget-button-top .beesender-widget-button-social {
    -webkit-box-orient: vertical;
    -webkit-box-direction: normal;
    -ms-flex-direction: column;
    flex-direction: column;
    -ms-flex-line-pack: start;
    align-content: flex-start
}

.beesender-widget-button-bottom .beesender-widget-button-inner-item {
    display: none
}

.beesender-widget-button-bottom .beesender-widget-button-close {
    display: block;
    -webkit-animation: socialRotate .4s;
    animation: socialRotate .4s;
    opacity: 1
}

.beesender-widget-button-top .beesender-widget-button-inner-item {
    display: none
}

.beesender-widget-button-top .beesender-widget-button-close {
    display: block;
    -webkit-animation: socialRotate .4s;
    animation: socialRotate .4s;
    opacity: 1
}

.beesender-widget-button-show {
    -webkit-animation: show .3s linear forwards;
    animation: show .3s linear forwards
}

@-webkit-keyframes show {
    0% {
        opacity: 0
    }

    50% {
        opacity: 0
    }

    100% {
        opacity: 1;
        visibility: visible
    }
}

@keyframes show {
    0% {
        opacity: 0
    }

    50% {
        opacity: 0
    }

    100% {
        opacity: 1;
        visibility: visible
    }
}

.beesender-widget-button-hide {
    -webkit-animation: hidden .3s linear forwards;
    animation: hidden .3s linear forwards
}

@-webkit-keyframes hidden {
    0% {
        opacity: 1;
        visibility: visible
    }

    50% {
        opacity: 1
    }

    99.999% {
        visibility: visible
    }

    100% {
        opacity: 0;
        visibility: hidden
    }
}

@keyframes hidden {
    0% {
        opacity: 1;
        visibility: visible
    }

    50% {
        opacity: 1
    }

    99.999% {
        visibility: visible
    }

    100% {
        opacity: 0;
        visibility: hidden
    }
}

.beesender-widget-button-hide-icons {
    -webkit-animation: hideIconsBottom .2s linear forwards;
    animation: hideIconsBottom .2s linear forwards
}

@-webkit-keyframes hideIconsBottom {
    0% {
        opacity: 1
    }

    50% {
        opacity: 1
    }

    100% {
        opacity: 0;
        -webkit-transform: translate(0, 20px);
        transform: translate(0, 20px);
        visibility: hidden
    }
}

@keyframes hideIconsBottom {
    0% {
        opacity: 1
    }

    50% {
        opacity: 1
    }

    100% {
        opacity: 0;
        -webkit-transform: translate(0, 20px);
        transform: translate(0, 20px);
        visibility: hidden
    }
}

@-webkit-keyframes hideIconsTop {
    0% {
        opacity: 1
    }

    50% {
        opacity: 1
    }

    100% {
        opacity: 0;
        -webkit-transform: translate(0, -20px);
        transform: translate(0, -20px);
        visibility: hidden
    }
}

@keyframes hideIconsTop {
    0% {
        opacity: 1
    }

    50% {
        opacity: 1
    }

    100% {
        opacity: 0;
        -webkit-transform: translate(0, -20px);
        transform: translate(0, -20px);
        visibility: hidden
    }
}

.beesender-widget-button-popup-name {
    font: bold 14px "Helvetica Neue", Arial, Helvetica, sans-serif;
    color: #000
}

.beesender-widget-button-popup-description {
    margin: 4px 0 0 0;
    font: 13px "Helvetica Neue", Arial, Helvetica, sans-serif;
    color: #424956
}

.beesender-widget-button-close-item {
    width: 28px;
    height: 28px;
    background-image: url('data:image/svg+xml;charset=US-ASCII,%3Csvg%20xmlns%3D%22http%3A//www.w3.org/2000/svg%22%20width%3D%2229%22%20height%3D%2229%22%20viewBox%3D%220%200%2029%2029%22%3E%3Cpath%20fill%3D%22%23FFF%22%20fill-rule%3D%22evenodd%22%20d%3D%22M18.866%2014.45l9.58-9.582L24.03.448l-9.587%209.58L4.873.447.455%204.866l9.575%209.587-9.583%209.57%204.418%204.42%209.58-9.577%209.58%209.58%204.42-4.42%22/%3E%3C/svg%3E');
    background-repeat: no-repeat;
    background-position: center;
    cursor: pointer
}

.beesender-widget-button-wrapper.beesender-widget-button-top {
    -webkit-box-orient: vertical;
    -webkit-box-direction: reverse;
    -ms-flex-direction: column-reverse;
    flex-direction: column-reverse
}

@-webkit-keyframes bottomOpen {
    0% {
        opacity: 0;
        -webkit-transform: translate(0, 20px);
        transform: translate(0, 20px)
    }

    100% {
        opacity: 1;
        -webkit-transform: translate(0, 0);
        transform: translate(0, 0)
    }
}

@keyframes bottomOpen {
    0% {
        opacity: 0;
        -webkit-transform: translate(0, 20px);
        transform: translate(0, 20px)
    }

    100% {
        opacity: 1;
        -webkit-transform: translate(0, 0);
        transform: translate(0, 0)
    }
}

@-webkit-keyframes topOpen {
    0% {
        opacity: 0;
        -webkit-transform: translate(0, -20px);
        transform: translate(0, -20px)
    }

    100% {
        opacity: 1;
        -webkit-transform: translate(0, 0);
        transform: translate(0, 0)
    }
}

@keyframes topOpen {
    0% {
        opacity: 0;
        -webkit-transform: translate(0, -20px);
        transform: translate(0, -20px)
    }

    100% {
        opacity: 1;
        -webkit-transform: translate(0, 0);
        transform: translate(0, 0)
    }
}

@-webkit-keyframes socialRotate {
    0% {
        -webkit-transform: rotate(-90deg);
        transform: rotate(-90deg)
    }

    100% {
        -webkit-transform: rotate(0deg);
        transform: rotate(0deg)
    }
}

@keyframes socialRotate {
    0% {
        -webkit-transform: rotate(-90deg);
        transform: rotate(-90deg)
    }

    100% {
        -webkit-transform: rotate(0deg);
        transform: rotate(0deg)
    }
}

@-webkit-keyframes socialRotateBack {
    0% {
        -webkit-transform: rotate(90deg);
        transform: rotate(90deg)
    }

    100% {
        -webkit-transform: rotate(0deg);
        transform: rotate(0deg)
    }
}

@keyframes socialRotateBack {
    0% {
        -webkit-transform: rotate(90deg);
        transform: rotate(90deg)
    }

    100% {
        -webkit-transform: rotate(0deg);
        transform: rotate(0deg)
    }
}

.beesender-widget-button-popup {
    display: none;
    position: absolute;
    left: 100px;
    padding: 12px 20px 12px 14px;
    width: 312px;
    border: 2px solid #2fc7f7;
    background: #fff;
    border-radius: 15px;
    box-sizing: border-box;
    z-index: 1;
    cursor: pointer
}

.beesender-widget-button-popup-triangle {
    position: absolute;
    display: block;
    width: 8px;
    height: 8px;
    background: #fff;
    border-right: 2px solid #2fc7f7;
    border-bottom: 2px solid #2fc7f7
}

.beesender-widget-button-popup-show {
    display: block;
    -webkit-animation: show .4s linear forwards;
    animation: show .4s linear forwards
}

.beesender-widget-button-position-top-left .beesender-widget-button-popup-triangle {
    top: 19px;
    left: -6px;
    -webkit-transform: rotate(134deg);
    transform: rotate(134deg)
}

.beesender-widget-button-position-bottom-left .beesender-widget-button-popup-triangle {
    bottom: 25px;
    left: -6px;
    -webkit-transform: rotate(134deg);
    transform: rotate(134deg)
}

.beesender-widget-button-position-bottom-left .beesender-widget-button-popup, .beesender-widget-button-position-bottom-middle .beesender-widget-button-popup {
    bottom: 0;
    left: 75px
}

.beesender-widget-button-position-bottom-right .beesender-widget-button-popup-triangle {
    bottom: 25px;
    right: -6px;
    -webkit-transform: rotate(-45deg);
    transform: rotate(-45deg)
}

.beesender-widget-button-position-bottom-right .beesender-widget-button-popup {
    left: -320px;
    bottom: 0
}

.beesender-widget-button-position-top-right .beesender-widget-button-popup-triangle {
    top: 19px;
    right: -6px;
    -webkit-transform: rotate(-45deg);
    transform: rotate(-45deg)
}

.beesender-widget-button-position-top-right .beesender-widget-button-popup {
    top: 0;
    left: -320px
}

.beesender-widget-button-position-top-middle .beesender-widget-button-popup-triangle {
    top: 19px;
    left: -6px;
    -webkit-transform: rotate(134deg);
    transform: rotate(134deg)
}

.beesender-widget-button-position-top-middle .beesender-widget-button-popup, .beesender-widget-button-position-top-left .beesender-widget-button-popup {
    top: 0;
    left: 75px
}

.beesender-widget-button-position-bottom-middle .beesender-widget-button-popup-triangle {
    bottom: 25px;
    left: -6px;
    -webkit-transform: rotate(134deg);
    transform: rotate(134deg)
}

.bx-touch .beesender-widget-button-popup {
    padding: 10px 22px 10px 15px
}

.bx-touch .beesender-widget-button-popup {
    width: 230px
}

.bx-touch .beesender-widget-button-position-bottom-left .beesender-widget-button-popup {
    bottom: 90px;
    left: 0
}

.bx-touch .beesender-widget-button-popup-image {
    margin: 0 auto 10px auto
}

.bx-touch .beesender-widget-button-popup-content {
    text-align: center
}

.bx-touch .beesender-widget-button-position-bottom-left .beesender-widget-button-popup-triangle {
    bottom: -6px;
    left: 25px;
    -webkit-transform: rotate(45deg);
    transform: rotate(45deg)
}

.bx-touch .beesender-widget-button-position-bottom-left .beesender-widget-button-popup {
    bottom: 90px;
    left: 0
}

.bx-touch .beesender-widget-button-position-bottom-right .beesender-widget-button-popup {
    bottom: 90px;
    left: -160px
}

.bx-touch .beesender-widget-button-position-bottom-right .beesender-widget-button-popup-triangle {
    bottom: -6px;
    right: 30px;
    -webkit-transform: rotate(-45deg);
    transform: rotate(45deg)
}

.bx-touch .beesender-widget-button-position-bottom-middle .beesender-widget-button-popup {
    bottom: 90px;
    left: 50%;
    -webkit-transform: translate(-50%, 0%);
    transform: translate(-50%, 0%)
}

.bx-touch .beesender-widget-button-position-bottom-middle .beesender-widget-button-popup-triangle {
    bottom: -6px;
    left: 108px;
    -webkit-transform: rotate(134deg);
    transform: rotate(45deg)
}

.bx-touch .beesender-widget-button-position-top-middle .beesender-widget-button-popup {
    top: 90px;
    left: 50%;
    -webkit-transform: translate(-50%, 0);
    transform: translate(-50%, 0)
}

.bx-touch .beesender-widget-button-position-top-middle .beesender-widget-button-popup-triangle {
    top: -7px;
    left: auto;
    right: 108px;
    -webkit-transform: rotate(-135deg);
    transform: rotate(-135deg)
}

.bx-touch .beesender-widget-button-position-top-left .beesender-widget-button-popup {
    top: 90px;
    left: 0
}

.bx-touch .beesender-widget-button-position-top-left .beesender-widget-button-popup-triangle {
    left: 25px;
    top: -6px;
    -webkit-transform: rotate(-135deg);
    transform: rotate(-135deg)
}

.bx-touch .beesender-widget-button-position-top-right .beesender-widget-button-popup {
    top: 90px;
    left: -150px
}

.bx-touch .beesender-widget-button-position-top-right .beesender-widget-button-popup-triangle {
    top: -7px;
    right: 40px;
    -webkit-transform: rotate(-135deg);
    transform: rotate(-135deg)
}

.beesender-widget-button-popup-btn-hide {
    position: absolute;
    top: 4px;
    right: 4px;
    display: inline-block;
    height: 20px;
    width: 20px;
    background-image: url('data:image/svg+xml;charset=US-ASCII,%3Csvg%20xmlns%3D%22http%3A//www.w3.org/2000/svg%22%20width%3D%2210%22%20height%3D%2210%22%20viewBox%3D%220%200%2010%2010%22%3E%3Cpath%20fill%3D%22%23525C68%22%20fill-rule%3D%22evenodd%22%20d%3D%22M6.41%205.07l2.867-2.864-1.34-1.34L5.07%203.73%202.207.867l-1.34%201.34L3.73%205.07.867%207.938l1.34%201.34L5.07%206.41l2.867%202.867%201.34-1.34L6.41%205.07z%22/%3E%3C/svg%3E');
    background-repeat: no-repeat;
    background-position: center;
    opacity: .2;
    -webkit-transition: opacity .3s;
    transition: opacity .3s;
    cursor: pointer
}

    .beesender-widget-button-popup-btn-hide:hover {
        opacity: 1
    }

.bx-touch .beesender-widget-button-popup-btn-hide {
    background-image: url('data:image/svg+xml;charset=US-ASCII,%3Csvg%20xmlns%3D%22http%3A//www.w3.org/2000/svg%22%20width%3D%2214%22%20height%3D%2214%22%20viewBox%3D%220%200%2014%2014%22%3E%3Cpath%20fill%3D%22%23525C68%22%20fill-rule%3D%22evenodd%22%20d%3D%22M8.36%207.02l5.34-5.34L12.36.34%207.02%205.68%201.68.34.34%201.68l5.34%205.34-5.34%205.342%201.34%201.34%205.34-5.34%205.34%205.34%201.34-1.34-5.34-5.34z%22/%3E%3C/svg%3E');
    background-repeat: no-repeat
}

.beesender-widget-button-popup-inner {
    display: -webkit-box;
    display: -ms-flexbox;
    display: flex;
    -ms-flex-flow: row wrap;
    flex-flow: row wrap
}

.beesender-widget-button-popup-content {
    width: 222px
}

.beesender-widget-button-popup-image {
    margin: 0 10px 0 0;
    width: 42px;
    text-align: center
}

.beesender-widget-button-popup-image-item {
    display: inline-block;
    width: 42px;
    height: 42px;
    border-radius: 100%;
    background-repeat: no-repeat;
    background-position: center;
    background-size: cover
}

.beesender-widget-button-popup-button {
    margin: 15px 0 0 0;
    -webkit-box-flex: 1;
    -ms-flex: 1;
    flex: 1;
    text-align: center
}

.beesender-widget-button-popup-button-item {
    display: inline-block;
    margin: 0 16px 0 0;
    font: bold 12px "Helvetica Neue", Arial, Helvetica, sans-serif;
    color: #08a6d8;
    text-transform: uppercase;
    border-bottom: 1px solid #08a6d8;
    -webkit-transition: border-bottom .3s;
    transition: border-bottom .3s;
    cursor: pointer
}

    .beesender-widget-button-popup-button-item:hover {
        border-bottom: 1px solid transparent
    }

    .beesender-widget-button-popup-button-item:last-child {
        margin: 0
    }

.beesender-widget-button-pulse {
    position: absolute;
    top: 0;
    left: 0;
    bottom: 0;
    right: 0;
    border: 1px solid #00aeef;
    border-radius: 50%
}

.beesender-widget-button-pulse-animate {
    -webkit-animation: widgetPulse infinite 1.5s;
    animation: widgetPulse infinite 1.5s
}

@-webkit-keyframes widgetPulse {
    50% {
        -webkit-transform: scale(1, 1);
        transform: scale(1, 1);
        opacity: 1
    }

    100% {
        -webkit-transform: scale(2, 2);
        transform: scale(2, 2);
        opacity: 0
    }
}

@keyframes widgetPulse {
    50% {
        -webkit-transform: scale(1, 1);
        transform: scale(1, 1);
        opacity: 1
    }

    100% {
        -webkit-transform: scale(2, 2);
        transform: scale(2, 2);
        opacity: 0
    }
}

@media(min-height:1024px) {
    .beesender-widget-button-top .beesender-widget-button-social, .beesender-widget-button-bottom .beesender-widget-button-social {
        max-height: 900px
    }
}

@media(max-height:768px) {
    .beesender-widget-button-top .beesender-widget-button-social, .beesender-widget-button-bottom .beesender-widget-button-social {
        max-height: 600px
    }
}

@media(max-height:667px) {
    .beesender-widget-button-top .beesender-widget-button-social, .beesender-widget-button-bottom .beesender-widget-button-social {
        max-height: 440px
    }
}

@media(max-height:568px) {
    .beesender-widget-button-top .beesender-widget-button-social, .beesender-widget-button-bottom .beesender-widget-button-social {
        max-height: 380px
    }
}

@media(max-height:480px) {
    .beesender-widget-button-top .beesender-widget-button-social, .beesender-widget-button-bottom .beesender-widget-button-social {
        max-height: 335px
    }
}

.altchat-team-profile-compact-team-name {
    padding-top: 8px !important;
}

#altchat-sendbox-textarea {
    padding-top: 9px !important;
}

.altchat-comment-container .altchat-comment-container-admi p {
    color: #3898ed !important;
}

a:hover {
    opacity: 1 !important;
}

#altchat-container .altchat-header-buttons-print {
    right: 44px !important;
    background-image: url(http://beesender.com/messanger/printer2.svg) !important;
    background-size: 20px !important;
}
    
</style>
