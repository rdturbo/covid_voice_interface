<template>
  <main id="app">
    <!-- The input -->
    <div class="query">
      <div class="wrapper" v-if="micro == false">
        <i class="material-icons iicon" @click="microphone(true)">mic</i>
        <input
          :aria-label="config.locale.strings.queryTitle"
          autocomplete="off"
          v-model="query"
          class="queryform"
          @keyup.enter="submit()"
          :placeholder="config.locale.strings.queryTitle"
          autofocus
          type="text"
        />
        <i class="material-icons iicon t2s" @click="mute(true)" v-if="muted == false">volume_up</i>
        <i class="material-icons iicon t2s" @click="mute(false)" v-else>volume_off</i>
      </div>
      <div class="wrapper" v-else>
        <i class="material-icons iicon recording" @click="microphone(false)">mic</i>
        <input class="queryform" :placeholder="speech" readonly />
      </div>
    </div>

    <section class="wrapper ai-window">
      <br />
      <br />

      <!-- Display Welcome Message -->
      <div v-if="answers.length == 0 && online == true">
        <h1 class="title mdc-typography--headline">
          <div class="material-icons up">arrow_upward</div>
          <br />
          <br />
          {{ config.locale.strings.welcomeTitle }}
          <p class="mdc-typography--body2">{{ config.locale.strings.welcomeDescription }}</p>
        </h1>
      </div>

      <!-- Display offline message -->
      <div v-if="answers.length == 0 && online == false">
        <h1 class="title mdc-typography--headline">
          <div class="material-icons up">cloud_off</div>
          <br />
          <br />
          {{ config.locale.strings.offlineTitle }}
          <p class="mdc-typography--body2">{{ config.locale.strings.offlineDescription }}</p>
        </h1>
      </div>

      <!-- Chat window -->
      <table v-for="(a, index) in answers" :key="index" class="chat-window">
        <!-- Your messages -->
        <tr>
          <td class="bubble">{{ a.result.resolvedQuery }}</td>
        </tr>

        <!-- Dialogflow messages -->
        <tr>
          <td>
            <!-- Bot message types / Speech -->
            <div
              v-if="a.result.fulfillment.speech"
              class="bubble bot"
            >{{ a.result.fulfillment.speech }}</div>

            <div
              v-if="
                a.result.action === 'input.unknown' &&
                  a.result.fulfillment.speech
              "
              class="google-chip chips"
            >
              <a
                class="suggestion"
                :href="
                  'https://www.google.com/search?q=' +
                    a.result.fulfillment.messages[0].text
                "
                target="_blank"
              >
                Search for "{{ a.result.fulfillment.messages[0].text }}" on
                Google
                <i
                  class="material-icons openlink"
                >search</i>
              </a>
            </div>
          </td>
        </tr>
      </table>

      <br />
      <a id="bottom"></a>
    </section>
  </main>
</template>

<style lang="sass">
@import url('https://fonts.googleapis.com/css?family=Roboto')
@import "App.sass"
</style>

<script>
import { ApiAiClient } from "api-ai-javascript";
import axios from "axios";
import moment from "moment";
import config from "./config";
const client = new ApiAiClient({ accessToken: config.app.token }); // <- replace it with yours
export default {
  name: "app",
  data: function() {
    return {
      answers: [],
      query: "",
      audioInstance: null,
      speech: config.locale.strings.voiceTitle,
      micro: false,
      muted: config.app.muted,
      online: navigator.onLine,
      synth: window.speechSynthesis,
      config
    };
  },
  watch: {
    answers: function() {
      setTimeout(() => {
        document.querySelector("#bottom").scrollIntoView({
          behavior: "smooth"
        });
      }, 2); // if new answers arrive, wait for render and then smoothly scroll down to #bottom selector, used as anchor
    }
  },
  methods: {
    submit() {
      client.textRequest(this.query).then(response => {
        if (response.result.action === "input.unknown") {
          response.result.fulfillment.messages[0].unknown = true;
          response.result.fulfillment.messages[0].text =
            response.result.resolvedQuery;
        } // if the googleIt is enabled, show the button
        this.changeResponse(response);
        //this.answers.push(response);
        //this.handle(response); // <- handle the response in handle() method
        //this.query = "";
        //this.speech = config.locale.strings.voiceTitle; // <- reset query and speech
      });
    },
    playMusic(response) {
      this.handleModifiedResponse(response);
      this.audioInstance = new Audio(require("./assets/pokemon.mp3"));
      this.audioInstance.play();
    },
    stopMusic(response) {
      if (this.audioInstance) {
        this.audioInstance.pause();
      }
      this.handleModifiedResponse(response);
    },
    getQueryString(type, latest) {
      switch (type) {
        case "confirmed":
          return `${latest.confirmed} confirmed cases`;
        case "deaths":
          return `${latest.deaths} deaths`;
        case "recovered":
          return `${latest.recovered} recovered cases`;
        case "all":
          return `${latest.confirmed} confirmed cases, ${latest.deaths} deaths `;
        default:
          return `${latest.confirmed} confirmed cases, ${latest.deaths} deaths, and ${latest.recovered} recovered cases`;
      }
    },
    handleModifiedResponse(modifiedResponse) {
      this.answers.push(modifiedResponse);
      this.handle(modifiedResponse); // <- handle the response in handle() method
      this.query = "";
      this.speech = config.locale.strings.voiceTitle; // <- reset query and speech
    },
    getModifiedResponse(dialogflowResponse, finalQuery) {
      return {
        ...dialogflowResponse,
        result: {
          ...dialogflowResponse.result,
          fulfillment: {
            ...dialogflowResponse.result.fulfillment,
            speech: finalQuery
          }
        }
      };
    },
    axiosCountryHandler(allData, dialogflowResponse, type) {
      let promises = [];
      let finalQuery = "";
      if (allData.length === 0) {
        const modifiedResponse = this.getModifiedResponse(
          dialogflowResponse,
          "Sorry, we didn't get the name of country. Can you repeat it again...?"
        );
        this.handleModifiedResponse(modifiedResponse);
        return;
      }
      const allCountry = allData.filter(el => el["alpha-2"] != "AD");
      allCountry.map((el, index) => {
        const countryCode = el["alpha-2"];
        promises.push(
          axios
            .get(
              `https://coronavirus-tracker-api.herokuapp.com/v2/locations?source=jhu&country_code=${countryCode}`
            )
            .then(res => {
              if (index === 1) {
                finalQuery += "In addition, there are ";
              } else {
                finalQuery += "There are ";
              }
              const { latest } = res.data;
              if (type.length >= 2) {
                const query = this.getQueryString("all", latest);
                finalQuery += `${query} in ${el.name}. `;
              } else {
                type.forEach(element => {
                  const query = this.getQueryString(element, latest);
                  finalQuery += `${query} in ${el.name}. `;
                });
              }
            })
            .catch(error => {
              console.log(error);
              finalQuery += " Some error occured, please try again later. ";
            })
        );
      });
      Promise.all(promises).then(() => {
        const modifiedResponse = this.getModifiedResponse(
          dialogflowResponse,
          finalQuery
        );
        this.handleModifiedResponse(modifiedResponse);
      });
    },
    axiosTimelineHandlerQuery(countryName, type, confirmed, death) {
      switch (type) {
        case "confirmed":
          return `There have been ${confirmed} confirmed cases in ${countryName}. `;
        case "deaths":
          return `There have been ${death} deaths in ${countryName}. `;
        case "recovered":
          return "Sorry we don't have any information for recovered cases ";
        case "all":
          return `There have been ${confirmed} confirmed cases, and ${death} deaths in ${countryName}. `;
        default:
          return `There have been ${confirmed} confirmed cases, ${death} deaths and 0 recovered cases in ${countryName} `;
      }
    },
    axiosTimelineHandler(type, date, country, dialogflowResponse) {
      let finalQuery = "";
      let promises = [];
      const allCountry = country.filter(el => el["alpha-2"] != "AD");
      if (!date || country.length === 0) {
        const modifiedResponse = this.getModifiedResponse(
          dialogflowResponse,
          "Sorry, we didn't get it. Can you say that again..?"
        );
        this.handleModifiedResponse(modifiedResponse);
        return;
      } else {
        const actualDate = date.split("/")[0];
        const endDate = actualDate + "T00:00:00Z";
        allCountry.map(cnt => {
          promises.push(
            axios
              .get(
                `https://coronavirus-tracker-api.herokuapp.com/v2/locations?source=jhu&country_code=${cnt["alpha-2"]}&timelines=true`
              )
              .then(res => {
                const { latest, locations } = res.data;
                const { confirmed, deaths } = locations[0].timelines;
                const confirmedSince =
                  latest.confirmed - confirmed.timeline[endDate];
                const deathSince = latest.deaths - deaths.timeline[endDate];
                if (type.length >= 2) {
                  const query = this.axiosTimelineHandlerQuery(
                    cnt.name,
                    "all",
                    confirmedSince,
                    deathSince
                  );
                  finalQuery += query;
                } else {
                  type.forEach(ty => {
                    const query = this.axiosTimelineHandlerQuery(
                      cnt.name,
                      ty,
                      confirmedSince,
                      deathSince
                    );
                    finalQuery += query;
                  });
                }
              })
              .catch(error => {
                console.log(error);
              })
          );
        });
        const humanDate = moment(actualDate).format("MMMM Do");
        Promise.all(promises).then(() => {
          const modifiedResponse = this.getModifiedResponse(
            dialogflowResponse,
            `Since ${humanDate}, ${finalQuery}`
          );
          this.handleModifiedResponse(modifiedResponse);
        });
      }
    },
    axiosTimelineCountyHandler(type, date, state, county, dialogflowResponse) {
      let finalQuery = "";
      let promises = [];
      if (!date || county.length === 0 || !state[0]) {
        const modifiedResponse = this.getModifiedResponse(
          dialogflowResponse,
          "Sorry, we didn't get it. Can you say that again..?"
        );
        this.handleModifiedResponse(modifiedResponse);
        return;
      } else {
        const actualDate = date.split("/")[0];
        const endDate = actualDate + "T00:00:00Z";
        county.map(cnt => {
          const countyString = cnt.toLowerCase().replace(" county", "");
          promises.push(
            axios
              .get(
                `https://coronavirus-tracker-api.herokuapp.com/v2/locations?source=nyt&province=${state[0]}&county=${countyString}&timelines=true`
              )
              .then(res => {
                const { latest, locations } = res.data;
                const { confirmed, deaths } = locations[0].timelines;
                const confirmedSince =
                  latest.confirmed - confirmed.timeline[endDate];
                const deathSince = latest.deaths - deaths.timeline[endDate];
                const modifiedCounty =
                  countyString.charAt(0).toUpperCase() +
                  countyString.slice(1) +
                  " County";
                if (type.length >= 2) {
                  const query = this.axiosTimelineHandlerQuery(
                    modifiedCounty,
                    "all",
                    confirmedSince,
                    deathSince
                  );
                  finalQuery += query;
                } else {
                  type.forEach(typ => {
                    const query = this.axiosTimelineHandlerQuery(
                      modifiedCounty,
                      typ,
                      confirmedSince,
                      deathSince
                    );
                    finalQuery += query;
                  });
                }
              })
              .catch(error => {
                console.log(error);
              })
          );
        });
        const humanDate = moment(actualDate).format("MMMM Do");
        Promise.all(promises).then(() => {
          const modifiedResponse = this.getModifiedResponse(
            dialogflowResponse,
            `Since ${humanDate}, ${finalQuery}`
          );
          this.handleModifiedResponse(modifiedResponse);
        });
      }
    },
    axiosCountyHandler(type, state, county, dialogflowResponse) {
      let promises = [];
      let finalQuery = "";
      if (!state[0] || county.length === 0) {
        const modifiedResponse = this.getModifiedResponse(
          dialogflowResponse,
          "I didn't get the name of the county. Can you say that again..?"
        );
        this.handleModifiedResponse(modifiedResponse);
        return;
      } else {
        county.map(el => {
          let countyString = el.toLowerCase();
          let countyPerish = "";
          if (countyString.includes("parish")) {
            countyString = countyString.replace(" parish", "");
            countyPerish = "Parish";
          } else {
            countyString = countyString.replace(" county", "");
            countyPerish = "County";
          }
          promises.push(
            axios
              .get(
                `https://coronavirus-tracker-api.herokuapp.com/v2/locations?source=csbs&province=${state[0]}&county=${countyString}`
              )
              .then(res => {
                const { latest } = res.data;
                if (type.length >= 2) {
                  const query = this.getQueryString("all", latest);
                  finalQuery += `There are ${query} in ${countyString
                    .charAt(0)
                    .toUpperCase() + countyString.slice(1)} ${countyPerish}. `;
                } else {
                  type.forEach(element => {
                    const query = this.getQueryString(element, latest);
                    finalQuery += `There are ${query} in ${countyString
                      .charAt(0)
                      .toUpperCase() +
                      countyString.slice(1)} ${countyPerish}. `;
                  });
                }
              })
              .catch(() => {
                finalQuery += "Some error occured, please try again later. ";
              })
          );
        });
      }
      Promise.all(promises).then(() => {
        const modifiedResponse = this.getModifiedResponse(
          dialogflowResponse,
          finalQuery
        );
        this.handleModifiedResponse(modifiedResponse);
      });
    },
    axiosStateHandler(allState, dialogflowResponse, type) {
      let promises = [];
      let finalQuery = "";
      if (allState.length === 0) {
        const modifiedResponse = this.getModifiedResponse(
          dialogflowResponse,
          "Sorry, I didn't get the state. Can you say that again..?"
        );
        this.handleModifiedResponse(modifiedResponse);
        return;
      }
      allState.map(el => {
        promises.push(
          axios
            .get(
              `https://coronavirus-tracker-api.herokuapp.com/v2/locations?source=csbs&province=${el}`
            )
            .then(res => {
              const { latest } = res.data;
              if (type.length >= 2) {
                const query = this.getQueryString("all", latest);
                finalQuery += `There are ${query} in ${el}. `;
              } else {
                type.forEach(element => {
                  const query = this.getQueryString(element, latest);
                  finalQuery += `There are ${query} in ${el}. `;
                });
              }
            })
            .catch(() => {
              finalQuery += "Some error occured, please try again later. ";
            })
        );
      });
      Promise.all(promises).then(() => {
        const modifiedResponse = this.getModifiedResponse(
          dialogflowResponse,
          finalQuery
        );
        this.handleModifiedResponse(modifiedResponse);
      });
    },
    changeResponse(response) {
      const { metadata } = response.result;
      if (metadata.intentName === "Country Latest Stats") {
        const { country, type } = response.result.parameters;
        this.axiosCountryHandler(country, response, type);
      } else if (metadata.intentName === "State Latest Stats") {
        const { state, type } = response.result.parameters;
        this.axiosStateHandler(state, response, type);
      } else if (
        metadata.intentName === "County Latest Stats" ||
        metadata.intentName === "Parish Latest Stats"
      ) {
        const { type, state, county } = response.result.parameters;
        this.axiosCountyHandler(type, state, county, response);
      } else if (metadata.intentName === "Country Time Stats") {
        const { type, date, country } = response.result.parameters;
        this.axiosTimelineHandler(type, date, country, response);
      } else if (metadata.intentName === "County Time Stats") {
        const { type, date, state, county } = response.result.parameters;
        this.axiosTimelineCountyHandler(type, date, state, county, response);
      } else if (metadata.intentName === "Play music") {
        this.playMusic(response);
      } else if (metadata.intentName === "Stop music") {
        this.stopMusic(response);
      } else {
        this.handleModifiedResponse(response);
      }
    },
    handle(response) {
      if (response.result.fulfillment.speech) {
        let text = response.result.fulfillment.speech.replace(
          /(?:https?|ftp):\/\/[\n\S]+/g,
          ""
        );
        const speech = new SpeechSynthesisUtterance(text);
        speech.voiceURI = "Google UK English Female";
        speech.rate = 1.2;
        speech.pitch = 0.8;
        // speech.lang = "en-US";
        if (!this.muted) {
          window.speechSynthesis.cancel();
          window.speechSynthesis.speak(speech); // <- if app is not muted, speak out the speech
          const timeout = setInterval(function() {
            if (!window.speechSynthesis.speaking) {
              clearInterval(timeout);
            } else {
              window.speechSynthesis.resume();
            }
          }, 14000);
        }
      }
    },
    autosubmit(suggestion) {
      this.query = suggestion;
      this.submit();
    },
    mute(mode) {
      this.muted = mode;
    },
    microphone(mode) {
      this.micro = mode;
      let self = this; // <- correct scope
      if (mode == true) {
        // eslint-disable-next-line no-undef
        let recognition = new webkitSpeechRecognition(); // <- chrome speech recognition
        recognition.interimResults = true;
        recognition.lang = config.locale.settings.recognitionLang;
        recognition.start();
        recognition.onresult = function(event) {
          for (var i = event.resultIndex; i < event.results.length; ++i) {
            self.speech = event.results[i][0].transcript;
          }
        };
        recognition.onend = function() {
          recognition.stop();
          self.micro = false;
          self.autosubmit(self.speech);
        };
      }
    }
  }
};
</script>