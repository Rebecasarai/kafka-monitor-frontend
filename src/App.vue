<template>
   <div id="app" class="container">
      <div class="columns">
         <div class="column">
            <h2 class="title is-2">Real time Kafka Monitor</h2>
            <div id="main" style="min-witdh:100%; min-height:400px; "></div>
         </div>
      </div>
      <div class="columns">
         <div class="column bottom-panel">
            <div id="trafficpanel" >
               <aside>
                  <h5 class="title is-5">Topics Speed</h5>
                  <div class="column jobstraffic-green" >
                     <h6 class="title is-6">Fast (Over average)</h6>
                     <ul class="menu-list is-size-7">
                        <li  v-for="topic in topicsSpeed.fast" :key="topic">
                           <p v-bind:id="topic">{{ topic }}</p>
                        </li>
                     </ul>
                  </div>
                  <div class="column  jobstraffic-yellow" >
                    <h6 class="title is-6">Slow (Under the average)</h6>
                    <ul class="menu-list is-size-7">
                      <li  v-for="topic in topicsSpeed.slow" :key="topic">
                          <p v-bind:id="topic">{{ topic }}</p>
                      </li>
                    </ul>
                  </div>
                  <div class="column is jobstraffic-red" >
                    <h6 class="title is-6">Stopped</h6>
                    <ul class="menu-list is-size-7">
                      <li v-for="topic in topicsSpeed.stopped" :key="topic">
                          <p v-bind:id="topic">{{ topic }}</p>
                      </li>
                    </ul>
                  </div>
               </aside>
            </div>
         </div>
         <div class="column bottom-panel">
               <h5 class="title is-5">Received messages</h5>
               <div class="select">
                  <select v-model="selectedTime">
                     <option value="1" selected>Segundos</option>
                     <option value="60">Minutos</option>
                     <option value="3600">Horas</option>
                  </select>
               </div>
               <div id="counter" >
                  <p id="counterp">Total received: {{ this.counter }}</p>
                  <p id="messagePerInterval">{{ this.messagesPerInterval }}
                     <span v-if="selectedTime == 1">/s</span>
                     <span v-else-if="selectedTime == 60">/m</span>
                     <span v-else>/h</span>
                  </p>
                  <p id="secondsspent"> {{ this.secondsSpent/this.selectedTime }} 
                     <span v-if="selectedTime==1">Seconds</span> 
                     <span v-else-if="selectedTime==60">Minutes</span>
                     <span v-else>Hours</span>
                  </p>
               </div>
            </div>
         <div class="column bottom-panel">
            <!--<div class="columns">-->
               <h5 class="title is-5" >Notifications</h5>
               <h3>When the average traffic is</h3>
               <input class="input" v-model.number="minimumTraffic" type="number" placeholder="2" value="2">
               <h3>In the topic/s</h3>
               <multiselect 
                  v-model="selectedTopics" 
                  :options="topicsNames"
                  :multiple="true"
                  >
               </multiselect>
               <h4>Notify me every:</h4>
               <input class="input" type="number" v-model="notificationsTime" placeholder="2" value="2">  
            
               <div class="select">
                  <select v-model="notificationsTimeMeasure">
                     <option value="1" >Seconds</option>
                     <option value="60" selected>Minutes</option>
                     <option value="3600">Hours</option>
                  </select>
               </div>
         </div>
      </div>
   </div>
</template>

<script>
/** 
 * Front end part of the kafka monitor
 * Using the reactive nature of Vue, it was build a version of a real time kafka monitor 
 * It recieves a message every certain time from the backend, through Socket methods. This interval perfectly configurable from "varconfig.js",
 * As well, as the cloudera hosts, topics and other data is so, from the "config.json"
 * For each topic, a representation in the chart is wshown 
 * It has multiselection panel to pick the topics to be notified when reach to a minimum traffic
 * 
 * Vue apps have a life lifecycle, in "mounted()", is posible to execute code once the app is ready and the template is loaded
 * 
 */
/* eslint-disable */
import echarts from 'echarts'
import {  flatMap, sum, max, keys, values, mapValues, map, mapKeys, has, reduce, some, remove, filter, find, findIndex, differenceBy  } from 'lodash'

import Vue from 'vue'
import VueNativeNotification from 'vue-native-notification'
import Multiselect from 'vue-multiselect'

export default {
  name: 'App',
  
  components: { Multiselect },
  data () {
    return {
      data: [],
      topicsNames: [],
      date: [],
      topics: [],
      counter: 0,
      messagesPerInterval: 0,
      secondsSpent: 0,
      selectedTime: 1,
      notificationsTime: 5,
      minimumTraffic: 0,
      notificationsTimeMeasure: 1,
      numeroSlice: -5,
      interval: 5000,
      selectedTopics: [],
      chartTopics: [],
      notiChange: false,
      timer: '',
      topicsSpeed: {fast: [], slow:[], stopped: [] },
      totalAverage: []
    }
  },

  /** @description Once the app is created this is executed 
   * */
  mounted () {
      Vue.use(VueNativeNotification, {
        requestOnNotify: true
      })

    this.countSeconds() // starts counting the seconds since the app charges

    this.createMultipleChart() // cerates the chart
    this.notifyMinimum() // to notifiy if a topic has reached a minimum traffic 
  },

  /**@description Socket functions, used to comunicate from backend
   * exabeat receives the messages with kafka topics increments data
   * exabeatTopicsChanged notifies in real time when config file is modified
   * 
  */
  sockets: {
    /**
     * @param {Object} data Represents the message from the server with the topics data */
    exabeat (data) {
      this.processData(data)
      this.setChartdata()
      this.calculateSpeed()
    },
    /**
     * @param {String} data Represents an action that ocurred from the backend, like changes 
     * on the config file */
    exabeatTopicsChanged(data){
      this.setNotification(data)  
    }
  },

  /**@description Watches for value changes on these  variables */
  watch:{
    notificationsTime: function(){
      clearInterval(this.timer)

    },
    notificationsTimeMeasure: function(){
      clearInterval(this.timer)
    }

  },
  methods: {
    /**@description Shows a notification with a title and a body message
     * @param {String} titulo Represents the header of the notification
     * @param {String} message represents the body message
     */
    showNotification(titulo, message){

      this.$notification.show(titulo, {
      body: message
      }, {})

    },
    /**@description Identifies what tipe of notification message should show based on the string action
     * Can be added more types of actions
     */
    setNotification(action){
      switch (action) {
        case 'changed':
          this.showNotification('Topics cambiados', 'Se han cambiado los topics desde el archivo de configuración')
          break
        case 'removed':
          this.showNotification('Se ha borrado el fichero de configuración', 'Archivo Config.json borrado')
          break
        default:
        this.showNotification('Se han realizado cambios sobre el fichero de configuración', 'Archivo Config.json cambiado')
          break
      }
    },
    /** 
     * @description Checks if a minimum traffic has ben reached
     */
    checkIfReached(x){
        return x <= this.minimumTraffic
    },

    /** 
     * @description Reduces chartTopics to a specific array to use from the selected topics
     * data for the minimum traffic notifications. If topic 1 and topic 2 is selected from 
     * the panel, then this function will reduce from the current global chartTopics to
     * only the thata of those selected topics
     * @returns {array} valuesTocheck, representing the array reduced  
     */
    reduceTopics(){

      var valuesToCheck = this.chartTopics.reduce((results, value) => {
        const nombre = value.name
        return (this.selectedTopics.indexOf(nombre) > -1) ? [...results, { value, name }] : results;
      }, []).map(value => value.value.data)

      return valuesToCheck[0]
    },

    /**
     * @description Notififies with a native notification when a minimum of increments of a topic has been reached 
     */
    notifyMinimum(){
      
      this.timer = setInterval(function () {

        if(some(this.reduceTopics(), this.checkIfReached)){
          this.showNotification('Topic ha llegado a '+ this.minimumTraffic,'Se ha llegado al trafico minimo de mensajes asignado: '+this.minimumTraffic)
        }
      }.bind(this), parseInt(this.notificationsTime * this.notificationsTimeMeasure) * 1000)
    },

    /**
     * @description counts the seconds since the web app was opened or refreshed 
     * 
     * */
    countSeconds(){
      setInterval(function () {
        this.secondsSpent++
      }.bind(this), 1000)
    },

    /**
     * @description Calculates the media of messages received from backend 
     * */
    calculateMessagesMedia(){
      this.counter++
      this.messagesPerInterval = parseFloat(this.counter / this.secondsSpent) * this.selectedTime
    },

    /**
     * @description Sets the options of the chart: Legend, xAxis and Series with reactive global variable:
     * this.topicsNames, this.date and this.chartTopics
     * */
    setChartdata(){
      this.multipleChart.setOption({
        legend: {
            data: this.topicsNames,
            type: 'scroll',
            left: 10
        },
        xAxis: {
          data: this.date
        },
         series : this.chartTopics
      })
    },

    /**@description Process the data passed by the exabeat socket function. 
     * This is a message sent from the backend and contains the KAFKA data
     * @param data 
     */
    async processData(data){
      this.calculateMessagesMedia()
      const now = new Date()
      const strDate = [now.getHours(), now.getMinutes()].join(':')

      this.data.push(data)
      this.date.push(strDate)
      this.data = this.data.slice(this.numeroSlice)
      this.date = this.date.slice(this.numeroSlice)
      this.interval = this.data[this.data.length-1].interval

      await this.organizeTopics()
    },

    /**@description Makes the chart dinamic, depending on the kafka topics information. 
     * Updates in real time, removing or adding the lines representing the topics traffic
     * Sets an array with the options data for the chart. This is mainly done beacuse when
     * a new topic is added, throws an error if the type or stack is not defined. s
     */
    async organizeTopics(){
      var tmp = []
      var dataTopics = this.data[this.data.length-1].topics

      for(let i in dataTopics){

        var topicFromMessage = dataTopics[i] // message of each topic with its increment

        if(this.topics[topicFromMessage.topicName]){ // if the array of topics has a key name node of the topic, i.e. it exists

          tmp[topicFromMessage.topicName] = Object.assign([], this.topics[topicFromMessage.topicName])
          tmp[topicFromMessage.topicName].push(topicFromMessage.increment)
          tmp[topicFromMessage.topicName] = tmp[topicFromMessage.topicName].slice(this.numeroSlice)

        } else {
          tmp[topicFromMessage.topicName] = this.firstTimeIncrements(topicFromMessage)
        }
      }

      this.topics = tmp
      this.topicsNames = keys(this.topics)

      this.checkChanges()
      this.chartTopics = values(mapValues(this.topics, (data, name) => {
        return {
          name: name, 
          type:'line',
          stack: 'increments',
          areaStyle: {normal: {}},  
          data: data 
        }
      }))
    },
    /**
     * Gets the difference between 
     */
    differenceByTopicName(){
      var keyMap = {
          topicName: 'name'
        }
      var y = this.data[this.data.length-1].topics.map(function(obj) {
        return mapKeys(obj, function(value, key) {
          return keyMap[key];
        })
      })
        
      return differenceBy(this.chartTopics, y, 'name')
    },
    /**
     * Deletes topics that have been removed from config file in the chart, by going 
     * through the differences loop and deleting each one.
     * @param {Object} differenc Represents the array resulting from the difference 
     * of The new Topics to display on the chart and the old ones.
     */
    deleteTopicsFromChart(differenc){
      for (let i = 0; i < differenc.length; i++) {
        var index = findIndex(this.chartTopics, function(t) { return t.name == differenc[i].name })
        
        if(index !== -1 && typeof this.chartTopics[index] !== 'undefined'){
          this.chartTopics[index].data = null
        }
      }
    },
    /**
     * @description checks if there has been changes on topic between the ne data that comes from server to the chart's one
     * If so, deletes the data of that or those topics from the chart. So it really updates real time
     */
    checkChanges(){

      var differenc = this.differenceByTopicName()
      if( differenc.length > 0){
        this.deleteTopicsFromChart(differenc)
        this.setChartdata()
      }
    },

    /**
     * @description Creates an array to fullfill the data in case a new topic is added. Adds 0 to the left of the data array
     * If a new topic is added so it shows corectly 
     * @param {Object} topicFromMessage Represents the topic with increments from the server message
     * */
    firstTimeIncrements(topicFromMessage){
      var firstTimeIncrements = []
      var maxLength = 0
      
      if(this.chartTopics.length > 0){
        maxLength = max( this.chartTopics, function(t){ return t.data.length })
        for (let index = 0; index < maxLength.data.length; index++) {
          firstTimeIncrements.push(0)
        }
      }
      firstTimeIncrements.push(topicFromMessage.increment)
      return firstTimeIncrements
    },

    /**@description creates the chart to visualize the topics increments, setting
     * the options of it, like tooltip, toolbox, etc */
    createMultipleChart(){

      // initialize echarts instance with prepared DOM
      this.multipleChart = echarts.init(document.getElementById('main'))
      
      this.multipleChart.setOption({ // Chart options is setted
      title: {
          text: ''
      },
      grid:{
        y2: 100
      },
      tooltip : {
        trigger: 'axis',
        axisPointer: {
            type: 'cross',
            label: {
                backgroundColor: '#6a7985'
        },
        formatter: function (params) {
            console.log(params)
          }
        }
      },
      toolbox: {
        show: true,
        bottom: 0,
        right: 80,
        feature: {
          dataZoom: {
            yAxisIndex: 'none',
            title: {
              zoom: 'Zoom',
              back: 'Back'
            }
          },
          magicType: {
            type: ['line', 'bar'],
            title: {
              line: 'Line chart',
              bar: 'Bar chart'
            }
          },
          restore: {
            title: 'Restore'
          },
          saveAsImage: {
            name: 'Save',
            title: 'Save'
          }
        }
      },
      grid: {
        left: '3%',
        right: '4%',
        bottom: '3%',
        containLabel: true
      },
      xAxis : [
        {
          type: 'category',
          boundaryGap: false
        }
      ],
      yAxis : [
        {
          type: 'value',
          boundaryGap: [0, '30%'],
          axisLabel: {
            formatter: '{value} req/'+ this.interval/1000 +'s'
          }
        }
      ]
    })

    /** @description for responsive design, recreates the chart each time the screen width is changed */
    this.$nextTick(() => {
      window.addEventListener('resize', () => {
        [this.multipleChart].forEach(c => {
          c.resize()
        })
      })
    })
        
    },
    getTopicNames(topics){
      return map(topics, topic => topic.name)
    },
    /**@description Calculates the average of increments in topics, assigning them the category based on the average.
     */
    calculateSpeed(){
      const incrementos = flatMap(this.chartTopics, 'data')
      this.totalAverage = sum(incrementos)/incrementos.length

      const stopped = filter(this.chartTopics, topic => sum(topic.data)/topic.data.length < 1)
      const slow = filter(this.chartTopics, topic => {
       const _sum  = sum(topic.data)/topic.data.length
       return _sum >=1 && _sum < this.totalAverage
      })
      const fast = filter(this.chartTopics, topic => sum(topic.data)/topic.data.length >= this.totalAverage)

      this.topicsSpeed.stopped = this.getTopicNames(stopped)
      this.topicsSpeed.slow = this.getTopicNames(slow)
      this.topicsSpeed.fast = this.getTopicNames(fast)
    }
  },
}
</script>

<style>
#app {
  margin-top: 30px
}
.jobstraffic-green{
  background-color: rgba(135, 216, 135, 0.49) !important; 
  border-top-left-radius: 5px;
  border-top-right-radius: 5px;
}

.jobstraffic-yellow{
  background-color: #fbfba1 !important;
}

.jobstraffic-red{
  background-color: #ef9090 !important;
  border-bottom-left-radius: 5px;
  border-bottom-right-radius: 5px;
}
.bottom-panel{
  margin: 1% 3%;
}
</style>





<style src="vue-multiselect/dist/vue-multiselect.min.css"></style>