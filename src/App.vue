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
                     <p>Fast</p>
                     <ul class="menu-list is-size-7">
                        <li  v-for="topic in topicsSpeed.fast" :key="topic">
                           <p v-bind:id="topic">{{ topic }}</p>
                        </li>
                     </ul>
                  </div>
                  <div class="column  jobstraffic-yellow" >
                    <p>Slow</p>
                    <ul class="menu-list is-size-7">
                      <li  v-for="topic in topicsSpeed.slow" :key="topic">
                          <p v-bind:id="topic">{{ topic }}</p>
                      </li>
                    </ul>
                  </div>
                  <div class="column is jobstraffic-red" >
                    <p>Stopped</p>
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
import { keys, values, mapValues, pickBy, map, has, reduce, some, remove, uniq  } from 'lodash'

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
      topicsSpeed: {fast: [], slow:[], stopped: [] }
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

    /**
     * @description Called when topics are removed from config file, to not show them on the chart
     * And refill the data array of that topic with 0s
     * If the global variable of topics doesn't have the same lenght, then go trough the topics
     * and if one of those topics name is not equal to the one receiving from the serve,
     * it fills its data array of increments to 0 and later on removes it
     */
    closeTopicLine(msg){          
      if(this.chartTopics.length !== this.data[0].topics.length){
          for (let index = 0; index < this.chartTopics.length; index++) {
            const element = this.chartTopics[index]
            if(element.name !== msg.topicName){
              
              for (let j = 0; j < 4; j++) {
                this.chartTopics[index].data.push(0)
              }
              this.chartTopics[index].data = this.chartTopics[index].data.slice(this.numeroSlice)
              // this.removeTopic(msg, index)
            }
          }
        this.setChartdata()
      }
    },

    /**
     * @description Removes a topic based on its index
     * 
    **/
    removeTopic(msg, index){

      this.chartTopics.splice(index, 1)
      console.log(this.chartTopics)
      this.chartTopics = this.chartTopics.slice(this.numeroSlice)
    },

    /**@description Process the data passed by the exabeat socket function. 
     * This is a message sent from the backend and contains the KAFKA data
     * @param data 
     */
    processData(data){

      this.calculateMessagesMedia()
      const now = new Date()
      const strDate = [now.getHours(), now.getMinutes()].join(':')

      this.data.push(data)
      this.date.push(strDate)
      this.data = this.data.slice(this.numeroSlice)
      this.date = this.date.slice(this.numeroSlice)
      this.interval = this.data[0].interval

      this.organizeTopics()
      
    },

    /**@description Makes the chart dinamic, depending on the kafka topics information. 
     * Updates in real time, removing or adding the lines representing the topics traffic
     * Sets an array with the options data for the chart. This is mainly done beacuse when
     * a new topic is added, throws an error if the type or stack is not defined. s
     */
    organizeTopics(){
      var tmp = []
      var dataTopics = this.data[0].topics

      for(let i in dataTopics){

          const msg = dataTopics[i] // message of each topic with its increment
          this.closeTopicLine(msg)

          if(this.topics[msg.topicName]){ // if the array of topics has a key name node of the topic, i.e. it exists

            tmp[msg.topicName] = Object.assign([], this.topics[msg.topicName])
            tmp[msg.topicName].push(msg.increment)
            tmp[msg.topicName] = tmp[msg.topicName].slice(this.numeroSlice)

          } else {
            tmp[msg.topicName] = this.firstTimeIncrements(msg)
          }
      }

      // console.log(JSON.stringify(keys(tmp))+'\n'+JSON.stringify(values(tmp)))
      this.topics = tmp
      this.topicsNames = keys(this.topics)

      this.chartTopics = values(mapValues(this.topics, (data, name) => {
        return {
          name: name, 
          type:'line',
          stack: '总量',
          areaStyle: {normal: {}},  
          data: data }
      }))
    },

    /**
     * @description Creates an array to fullfill the data in case a new topic is added
     * 
     * */
    firstTimeIncrements(msg){
      var firstTimeIncrements = []

      for (let index = 0; index < Math.abs(this.numeroSlice); index++) {
        firstTimeIncrements.push(0)
      }
      firstTimeIncrements.push(msg.increment)
      return firstTimeIncrements
    },

    /**@description creates the chart to visualize the topics increments 
     * 
    */
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
      /**@description Calculates the average
       * @param {int array} increments Represents the array of increments of a particular topic
       */
      calculateAverage(increments){
        var sum = 0
        for( var i = 0; i < increments.length; i++ ){
            sum += parseInt( increments[i], 10 ) // add the base
        }

        return sum/increments.length
      },
      /**@description Calculates the average of increments in topics, assigning them the category based on the average.
       */
      calculateSpeed(){
        for (let index = 0; index < this.chartTopics.length; index++) {
          const increments = this.chartTopics[index].data
          var avg = this.calculateAverage(increments)
          this.asignSpeed(avg, index)
        }
      },
      /**
       * @description Determines the category of speed of a topic based on its average increments
       */
      asignSpeed(avg, index){
        if(avg < 1){

            this.topicsSpeed.stopped.push(this.chartTopics[index].name)
            this.topicsSpeed.stopped = uniq(this.topicsSpeed.stopped)
            this.validateTopicSpeed('stopped', index)

          }else if(avg < 10){

            this.topicsSpeed.slow.push(this.chartTopics[index].name)
            this.topicsSpeed.slow = uniq(this.topicsSpeed.slow)
            this.validateTopicSpeed('slow', index)
           
          }else{
            this.topicsSpeed.fast.push(this.chartTopics[index].name)
            this.topicsSpeed.fast = uniq(this.topicsSpeed.fast)
            this.validateTopicSpeed('fast', index)
        }
      },

      /**
       * @description Validates that if a topic is slow, then is not fast or stopped anymore.
       * Its necessary to remove from other speed arrays 
       * @param {String} type  Rpresents the type of speed is now belonging to
       * @param {int} index  Represents the index of the global chartTopics array*/
      validateTopicSpeed(type, index){
        switch (type) {
          case 'fast':
            var i = this.topicsSpeed.slow.indexOf(this.chartTopics[index].name)
            var j = this.topicsSpeed.stopped.indexOf(this.chartTopics[index].name)

            if (i !== -1) this.topicsSpeed.slow.splice(i, 1)
            if (j !== -1) this.topicsSpeed.stopped.splice(j, 1)
            break

          case 'slow':
            var i = this.topicsSpeed.fast.indexOf(this.chartTopics[index].name)
            var j = this.topicsSpeed.stopped.indexOf(this.chartTopics[index].name)

            if (i !== -1) this.topicsSpeed.fast.splice(i, 1)
            if (j !== -1) this.topicsSpeed.stopped.splice(j, 1)
            
            break

          case 'stopped':
            var i = this.topicsSpeed.fast.indexOf(this.chartTopics[index].name)
            var j = this.topicsSpeed.slow.indexOf(this.chartTopics[index].name)

            if (i !== -1) this.topicsSpeed.fast.splice(i, 1)
            if (j !== -1) this.topicsSpeed.slow.splice(j, 1)
            break
        
          default:

            break
        }
      }
    },
    computed: { 
      fastTopics: function() {
        return pickBy(this.topics, function(t) {
          return t.data
        })
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
  border-radius: 5px;
}

.jobstraffic-yellow{
  background-color: #fbfba1 !important;
  border-radius: 5px;
}

.jobstraffic-red{
  background-color: #ef9090 !important;
  border-radius: 5px;
}
.bottom-panel{
  margin: 1% 3%;
}


</style>





<style src="vue-multiselect/dist/vue-multiselect.min.css"></style>