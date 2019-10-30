<template>
  <v-app>
    <v-content>
      <v-container>
        <v-row>
          <v-col class="col-12">
            <p class="display-4 justify-center d-none d-sm-flex">Teostokeikat</p>
            <p class="display-2 justify-center d-flex d-sm-none">Teostokeikat</p>
            <p
              class="font-weight-light font-italic text-center"
            >Kaikki vuoden 2014-2017 Teostolle ilmoitetut keikat</p>
          </v-col>
        </v-row>
        <v-row>
          <v-col class="col-12">
            <v-card class="mx-auto" max-width="644">
              <v-form @submit.prevent="submitForm">
                <v-card-text>
                  <v-text-field v-model="place" placeholder="Syötä kaupunki"></v-text-field>
                  <v-select v-model="year" :items="years" label="Vuosi"></v-select>
                  <v-btn
                    type="submit"
                    :disabled="(place == null || ready == 'loading')"
                    @click="curPage=1,search(curPage)"
                    large
                    color="primary"
                  >Etsi</v-btn>
                </v-card-text>
              </v-form>
            </v-card>
            <v-row>
              <v-col class="col-12">
                <div class="text-center">
                  <v-pagination
                    v-if="pages != 0"
                    :disabled="(ready == 'loading')"
                    @change="changePage($event)"
                    v-model="curPage"
                    :length="pages"
                  ></v-pagination>
                </div>
              </v-col>
            </v-row>
          </v-col>
        </v-row>
        <v-row>
          <v-col class="col-12">
            <div class="text-center">
              <p v-if="error == '404'">Antamillasi hakuehdoilla ei löytynyt mitään!</p>
              <v-progress-circular
                v-if="ready == 'loading'"
                :size="50"
                color="primary"
                indeterminate
              ></v-progress-circular>
            </div>

            <v-simple-table v-if="ready == true">
              <template v-slot:default>
                <thead>
                  <tr>
                    <th class="text-left">Tapahtuman nimi</th>
                    <th class="text-left">Paikka</th>
                    <th class="text-left">Päivämäärä</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="event in eventDetails" :key="event.id">
                    <td>{{ event.name }}</td>
                    <td>{{ event.venue.name }}</td>
                    <td>{{ event.startDate }}</td>
                  </tr>
                </tbody>
              </template>
            </v-simple-table>
          </v-col>
        </v-row>
      </v-container>
    </v-content>
    <v-footer color="grey lighten-5">
      <div>
        Data from &nbsp;
        <a
          href="http://api.teosto.fi/"
          target="_blank"
          rel="noopener noreferrer"
        >Teosto API</a>
      </div>
      <v-spacer></v-spacer>
      <div>Janne Karppinen &copy; {{ new Date().getFullYear() }}</div>
    </v-footer>
  </v-app>
</template>

<script>
import axios from 'axios';

export default {
  mounted: function() {},
  name: 'App',
  components: {},
  data: () => ({
    curPage: 1,
    pages: 0,
    limit: 15,
    ready: false,
    place: null,
    events: [],
    eventDetails: [],
    showDetails: [],
    year: '2014',
    years: ['2014', '2015', '2016', '2017'],
    error: null
  }),
  computed: {},
  watch: {
    place(newValue) {
      if (!newValue) this.place = null;
      if (newValue) this.error = null;
    },
    curPage(newValue) {
      this.search(newValue);
      console.log('Page Changed to: ' + newValue);
    }
  },
  methods: {
    changePage: function(newPage) {
      if (newPage.target.type == 'button') {
        this.curPage = parseInt(newPage.target.innerHTML);
      }
    },
    search: function(page) {
      this.ready = 'loading';
      this.events = [];
      this.eventDetails = [];
      axios
        .get(
          'http://api.teosto.fi/' +
            this.year +
            '/municipality?name=' +
            this.place +
            '&method=events&limit=' +
            this.limit +
            '&page=' +
            page
        )
        .then(res => {
          //get event data
          this.curPage = res.data.response_meta.page;
          this.pages = res.data.response_meta.pages;
          this.events = res.data.events;
          console.log('Fetching...');
          return this.getEventDetails();
        })
        .then(res => {
          //get event details data
          console.log('READY');
          res.forEach(event => {
            if (event.data.event.name == '<n/a>')
              event.data.event.name = 'NIMETÖN';
            var date = new Date(event.data.event.startDate);
            var realDate =
              date.getDate() +
              '.' +
              (date.getMonth() + 1) +
              '.' +
              date.getFullYear();
            event.data.event.startDate = realDate;
            this.eventDetails.push(event.data.event);
          }),
            // set time out to disallow user to fetch data repeatedly due to rate limit in API
            setTimeout(() => {
              this.ready = true;
            }, 1000);
        })
        .catch(error => {
          if (!error.response) {
            // network error
          } else {
            const code = error.response.status;
            const response = error.response.data;
            console.log('Error: ' + code);
            if (code == '404') (this.ready = false), (this.place = null);
            this.error = code;
          }
        });
    },
    getEventDetails: function() {
      let promisedEvents = [];
      this.events.forEach(event => {
        promisedEvents.push(axios.get(event.url));
      });

      return Promise.all(promisedEvents);
    }
  }
};
</script>
