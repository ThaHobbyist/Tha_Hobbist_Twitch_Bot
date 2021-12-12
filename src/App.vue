<template>
  <!-- <button v-if="!start" @click="startRec" class="btn">Start</button>
  <button v-if="start" @click="pauseRec" class="btn">Pause</button>
  <button v-if="start" @click="stopRec" class="btn">Stop</button> -->
  <div class="text_box"><p class="text"> {{ text }} </p></div>
</template>

<script>
import dotenv from 'dotenv';
dotenv.config()

import { ChatClient } from 'twitch-chat-client';
import { StaticAuthProvider } from 'twitch-auth';

const table = require('../bcp47.json');
const subtags = table.subtags

import translate from 'translate';
translate.engine = "google";
translate.key = process.env.VUE_APP_RAPID_API_KEY;

var SpeechRecognition = SpeechRecognition || webkitSpeechRecognition;
var SpeechGrammarList = SpeechGrammarList || webkitSpeechGrammarList;

var grammar = '#JSGF V1.0;'

var final_text = ""

var recognition = new SpeechRecognition();
var speechRecognitionList = new SpeechGrammarList();
speechRecognitionList.addFromString(grammar, 1);
recognition.grammars = speechRecognitionList;

recognition.interimResults = false;
recognition.continuous = true;


export default {
  name: 'App',
  data() {
    return {
      text: '',
      client: null,
      start: false,
      clientId: null,
      oAuthToken: null,
      authProvider: null,
      
    }
  },
  mounted() {

    this.main();

    
  },
  methods: {
    startRec() {
      console.log(this.start)
      if (this.start === true){
        recognition.abort();
        this.start = false;
      }
      this.start = !this.start;
      recognition.start();

    },
    pauseRec() {
      this.start = !this.start;
      recognition.stop();
    },
    stopRec() {
      this.start = !this.start;
      recognition.abort();
    },
    async translateString(str, frm) {
      const a = await translate(str, { from: frm});
      console.log('translated text:' + a);
      this.text = a;
    },
    tag(lang) {
      for (let i=0; i<subtags.length; i++) {
        if (lang.toLowerCase() == subtags[i].description[0].toLowerCase()){
          return subtags[i].subtag;
        }
      }
    },
    async main() {
      const regexpCommand = new RegExp(/^!([a-zA-Z0-9]+)(?:\W+)?(.*)?/);
      const commands= {
        help : {
          response: 'Type !translate <language name> and you will see english subtitles to the language that the streamer is speaking!'
        },
        translate : {
          response: (lang) => {
            `translating from ${lang} to english`;
            recognition.lang = this.tag(lang);
            this.startRec();
            console.log(lang);
            recognition.addEventListener('result', (event) => {
              var last = event.results.length - 1;
              final_text = event.results[last][0].transcript;
              console.log(final_text)
              this.translateString(final_text, lang);

            })
          }
        }
      }

      this.clientId = process.env.VUE_APP_TWITCH_CLIENT_ID;
      this.oAuthToken = process.env.VUE_APP_TWITCH_OAUTH_TOKEN;

      this.authProvider = new StaticAuthProvider(this.clientId, this.oAuthToken);
      this.client = new ChatClient(this.authProvider, { channels: [ 'Tha_Hobbist' ]});

      await this.client.connect();

      this.client.onMessage((channel, user, message) => {
        const isNotBot = user.toLowerCase() !== process.env.VUE_APP_TWITCH_BOT_USERNAME.toLowerCase();

        if (!isNotBot) return;

        const [raw, command, argument] = message.match(regexpCommand);

        // this.client.say(channel, ` ${user} used ${command} command with ${argument} arguments `);
        if (!(command in commands)) return;
        else if (command === 'help') this.client.say(channel, 'Type !translate <language name> and you will see english subtitles to the language that the streamer is speaking!');
        else if (command === 'translate') {
          this.client.say(channel, 'Starting Translation....');
          recognition.lang = this.tag(argument);
            this.startRec();
            console.log(argument);
            recognition.addEventListener('result', (event) => {
              var last = event.results.length - 1;
              final_text = event.results[last][0].transcript;
              console.log(final_text)
              this.translateString(final_text, argument);
            });
        };
        
        
      });
    }
  },
}
</script>

<style>
.text_box{
  display: flex;
  align-content: center;
  justify-content: center;
}

.text{
  font-family: 'Bangers', cursive;
  font-size: 40px;
  color: white;
}
</style>
