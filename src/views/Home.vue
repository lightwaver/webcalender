<template>
  <div class="legendContainer">
    Legende:
    <span v-for="(l, ix) in labels" :class="['legend', l.css]">{{ l.label }}</span>
  </div>
  <!-- https://antoniandre.github.io/vue-cal/?ref=madewithvuejs.com -->
  <v-progress-linear indeterminate v-if="loading > 0" color="#903035" height="5"></v-progress-linear>

<!-- 
  <style v-for=""></style>
-->

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
import { clearScreenDown } from "readline";

declare interface EventEntry {
  start: Date | string;
  end: Date | string;
  title: String;
  content: String;
  class: string;
  allDay: boolean;
}
declare interface CalConfigEntry {
  id: string,
  description: string,
  cssclass: string,
  showDescription: boolean,
  timeless: boolean,
  icalSrc: string,
  color: string,
}
declare interface Configuration {
  CORSProxy: string;
  calenderSources: CalConfigEntry[];
}

const showDialog = ref(false);
const events = ref<EventEntry[]>([]);
const selectedEvent = ref<any>();


if (!window.setImmediate || !setImmediate) {
  window.setImmediate = <any>((x: () => void) => {
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
let corsUrl = "";

async function getCal(calUrl: string, cssclass: string, nodescription: boolean = false, timeless: boolean = false, color?: string) {

  if (corsUrl) calUrl = corsUrl + encodeURIComponent(calUrl);
  const icalevents = await ical.async.fromURL(calUrl);

  if (color) {
    dynCssClasses.value.push({name: "dc" + dynCssClasses.value.length, color});
    cssclass += " dc" + dynCssClasses.value.length;
  }

  for (const entry of Object.values(icalevents)) {
    const event: any = entry;
    if (!event.summary) continue;

    const content = nodescription ? "" : event.description;

    let allDay = (new Date(event.start).getHours() == 0 && new Date(event.end).getHours() == 0);
    if (event.type === 'VEVENT' && event.rrule) {
      const dates = event.rrule.between(addDays(new Date(), -21), addDays(new Date(), 256))
      if (dates.length === 0) { continue; }
      const diff = event.end.valueOf() - event.start.valueOf();

      dates.forEach((date: Date) => {
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
}
const dynCssClasses = ref<{ name:string, color: string }[]>([]);
const labels = ref<{ label: string, css: string }[]>([]);
const errors = ref<string[]>([]);
const loading = ref(1);
fetch("configuration.json").then(async (result) => {
  const config = (await result.json()) as Configuration;
  corsUrl = config.CORSProxy;
  loading.value += config.calenderSources.length;
  config.calenderSources.forEach(cal => {
    getCal(cal.icalSrc, cal.cssclass, !cal.showDescription, cal.timeless, cal.color)
      .then(() => {
        labels.value.push({ label: cal.description, css: cal.cssclass });
        loading.value--;
      })
      .catch(e => {
        errors.value.push("error loading calender:" + cal.description + " - " + e);
        loading.value--;
      })
  })
  loading.value--;
});
</script>
<style>
.wiwoe {
  background-color: #FBBB21;
  color: black;
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
  color: black;
}

.heim {
  background-color: #9E69AF;
  color: white;
}

.bus {
  background-color: lightgrey;
  color: black;
}

.feiertag {
  background-color: lightsalmon;
  color: black;
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
  display: inline-block;
  white-space: nowrao;
  padding: 3px 10px;
}

.vuecal--years-view .vuecal__cell-content, .vuecal--year-view .vuecal__cell-content, .vuecal--month-view .vuecal__cell-content {
    justify-content: start;
}
</style>
