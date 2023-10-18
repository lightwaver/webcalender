<template>
  <div class="legendContainer">
    Legende:
  <span class="legend wiwoe">Wichtel & Wölflinge (WiWö)</span>
  <span class="legend gusp">Guides & Späher (GuSp)</span>
  <span class="legend caex">Caravalles & Explorer (CaEx)</span>
  <span class="legend raro">Ranger & Rover</span>
  <span class="legend erat">Elternrat und Gilde</span>
  <span class="legend heim">Heimvermietung </span>
  <span class="legend bus">Busvermietung</span>
  <span class="legend feiertag">Feiertag</span>
  <span class="legend ferien">Ferien</span>
</div>
  <!-- https://antoniandre.github.io/vue-cal/?ref=madewithvuejs.com -->
  <VueCal locale="de" :disableViews="['years', 'year']" overlaps-per-time-step :time-step="120" today-button
    :events="events" :time-from="8 * 60" :time-to="22 * 60" events-on-month-view="short" showAllDayEvents
    :on-event-click="onEventClick" active-view="month">
    <template #event="{ event, view }">
      <span class="vuecal__event-title" v-html="event.title" />
      <br v-if="view.id !== 'month'" />
      <small class="vuecal__event-time" v-if="!event.allDay && view === 'month'">
        <span>{{ event.start.formatTime("HH:mm") }}-{{ event.end.formatTime("HH:mm") }}</span>
      </small>

    </template>
  </VueCal>

  <!-- Using Vuetify (but we prefer Wave UI 🤘) -->
  <v-dialog v-model="showDialog" width="500">
    <v-card>
      <v-card-title :class="selectedEvent.class">
        <v-icon>mdi-calendar</v-icon>
        <span>{{ selectedEvent.title }}</span>
        <v-spacer />
        <strong>{{ selectedEvent.start && selectedEvent.start.format("DD.MM.YY") }}</strong>
      </v-card-title>
      <v-card-text>
        <p v-html="selectedEvent.contentFull" />
        <p>{{ selectedEvent.description }}</p>
        <ul>
          <li>Start: {{ selectedEvent.start && selectedEvent.start.formatTime() }}</li>
          <li>Ende: {{ selectedEvent.end && selectedEvent.end.formatTime() }}</li>
        </ul>
      </v-card-text>
      <v-card-actions>
        <v-btn @click="showDialog = false">OK</v-btn>
      </v-card-actions>
    </v-card>

  </v-dialog>
</template>

<script lang="ts" setup>
import { ref, reactive } from "vue";

import VueCal from "vue-cal";
import HelloWorld from '@/components/HelloWorld.vue';
import 'vue-cal/dist/vuecal.css';
import ical from 'node-ical';
import { setImmediate } from 'timers';

declare interface EventEntry {
  start: Date | string;
  end: Date | string;
  title: String;
  content: String;
  class: string;
  allDay: boolean;
}
const showDialog = ref(false);
const events = ref<EventEntry[]>([]);
const selectedEvent = ref<any>();

if (!window.setImmediate || !setImmediate) {
  window.setImmediate = <any>((x: () =>void) =>{
    console.log("ups setImmediate called");
    x();
  });
}

function onEventClick(event: any, e: Event) {
  selectedEvent.value = event
  showDialog.value = true

  // Prevent navigating to narrower view (default vue-cal behavior).
  e.stopPropagation()
}

//const events: { start: Date, end: Date, title: String }[] = [];
function addDays(dateOld: Date, days: number) {
  const date = new Date(dateOld.valueOf());
  const result = new Date(date);
  result.setDate(result.getDate() + days);
  return result;
};

function removeTime(date: Date) {
  return date.toISOString().substring(0, 10);
}

function getCal(calUrl: string, cssclass: string, nodescription: boolean = false, timeless: boolean = false) {
  ical.async.fromURL("https://corsproxy.io/?" + encodeURIComponent(calUrl))
    .then(icalevents =>{
      for (const entry of Object.values(icalevents)) {
        const event: any = entry;
        if (!event.summary) continue;

        const content = nodescription ? "" : event.description;

        let allDay = (new Date(event.start).getHours() == 0 && new Date(event.end).getHours() == 0);
        if (event.type === 'VEVENT' && event.rrule) {
          const dates = event.rrule.between(addDays(new Date(), -21), addDays(new Date(), 256))
          if (dates.length === 0) { continue; }
          const diff = event.end.valueOf() - event.start.valueOf();

          dates.forEach((date: Date) =>{
            let start: Date | string = date;
            let end: Date | string = new Date(date.valueOf() + diff);

            if (timeless) {
              start = end = removeTime(date);
              allDay = true;
            }
            events.value.push({
              start,
              end,
              title: event.summary,
              class: cssclass,
              content,
              allDay
            });
          });
        } else {
          let start = event.start;
          let end = event.end;

          if (timeless) {
            start = end = removeTime(start);
            allDay = true;
          }

          events.value.push({
            start: event.start,
            end: event.end,
            title: event.summary,
            class: cssclass,
            content,
            allDay
          });
        }
      }
    });
}

getCal("https://www.feiertage-oesterreich.at/content/kalender-download/force-download.php", "feiertag", true, true);
getCal("https://www.feiertage-oesterreich.at/content/kalender-download/force-download-school-holidays.php?region=N%C3%96", "ferien", true, true);

getCal("https://calendar.google.com/calendar/ical/a099e00543b616fed2ae3d680ff04dbd76aed8dccc8c7e796a198bea706c9b2a%40group.calendar.google.com/public/basic.ics",
  "wiwoe");
getCal("https://calendar.google.com/calendar/ical/1adf19db83284aaa0bfb8fed2c0f72e44ce94e53a75c0f202fc31cdba9d8ab4e%40group.calendar.google.com/public/basic.ics",
  "gusp");
getCal("https://calendar.google.com/calendar/ical/8609cf54b769090a50ef056878d6697595811eb24b2e486467d3fbd7330960fc%40group.calendar.google.com/public/basic.ics",
  "caex");
getCal("https://calendar.google.com/calendar/ical/54f2c3b56798e27a293a4c63ac5234abf60d6fd37a4a9ea6a2e86a95d91ce206%40group.calendar.google.com/public/basic.ics",
  "raro");
getCal("https://calendar.google.com/calendar/ical/82552bba125f828a36c4e802941af3141b52fb4193850e0cba539a23de09b0c5%40group.calendar.google.com/public/basic.ics",
  "erat");
getCal("https://calendar.google.com/calendar/ical/c8cb6dfd28d73a3cf1814ea52f4826c28ba28e5f1543a78111f3be5c6d5c07ea%40group.calendar.google.com/public/basic.ics",
  "heim");
getCal("https://calendar.google.com/calendar/ical/4983015c31103a16024f2fb13a21265d8cee6d2787409610560119528227e055%40group.calendar.google.com/public/basic.ics",
  "bus");
</script>
<style>
.wiwoe {
  background-color: #FBBB21;
  color:black;
}

.gusp {
  background-color: #159A34;
  color: white;
}

.caex {
  background-color: #0B4697;
  color: white;
}

.raro {
  background-color: #E62336;
  color: white;
}

.erat {
  background-color: #A6E1FF;
  color:black;
}

.heim {
  background-color: #9E69AF;
  color: white;
}

.bus {
  background-color: lightgrey;
  color:black;
}

.feiertag {
  background-color: lightsalmon;
  color:black;
}

.ferien {
  background-color: lightseagreen;
  color: black
}

.vuecal__event {
  padding-top: 3px;
  line-height: 1em;
  cursor: pointer;
}
.legendContainer {
  text-align: center;
}
.legend {
  display:inline-block;
  white-space: nowrao;
  padding: 3px 10px;
}
</style>
