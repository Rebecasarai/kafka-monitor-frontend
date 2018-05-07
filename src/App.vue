<template>
  <div id="app" class="container">
    <div class="columns">

      <div class="column">
        <h2 class="title is-2">Kafka Monitor</h2>
        <div id="main" style="min-witdh:100%; min-height:400px; "></div>
        <div></div>
        <div></div>
        <div></div>
      </div>
    </div>


    <div class="columns">
      <div class="column">
        <div id="trafficpanel" >
          <aside>
            <h5 class="title is-5">Diferencias de mensajes</h5>
              <div class="column is-half jobstraffic-green" style="background-color:#87d887 !important;">
                <p>Rapidos</p>
                <ul class="menu-list is-size-7">
                  <li v-for="topic in topicsNames" :key="topic">
                    <p v-bind:id="topic">{{ topic }}</p>
                  </li>
                </ul>
              </div>
              <div class="column is-half jobstraffic-yellow" style="background-color:#fbfba1 !important;">Lentos</div>
              <div class="column is-half jobstraffic-red" style="background-color:#ef9090 !important;">Parados</div>
          </aside>
        </div>
      </div>

      <div class="column">
        <div class="column">
          <h5 class="title is-5">Velocidad promedio</h5>
          <div class="select">
            <select v-model="selectedTime">
              <option value="1" selected>Segundos</option>
              <option value="60">Minutos</option>
              <option value="3600">Horas</option>
            </select>
          </div>
        </div>
        <div class="column">
          <div id="counter" >
            <p id="counterp">Total: {{ this.counter }}</p>
            <p id="messagePerInterval">{{ this.messagesPerInterval }}
              <span v-if="selectedTime == 1">/s</span>
              <span v-else-if="selectedTime == 60">/m</span>
              <span v-else>/h</span>
            </p>
            <p id="secondsspent"> {{ this.secondsSpent/this.selectedTime }} 
              <span v-if="selectedTime==1">Segundos</span> 
              <span v-else-if="selectedTime==60">Minutos</span>
              <span v-else>horas</span></p>
          </div>

        </div>
      </div>

      <div class="column">
        

        <!--<div class="columns">-->
        <div class="column">
        
        <h5 class="title is-5" >Notificaciones</h5>

            <h3>Cuando la media del trafico sea</h3>
            <input class="input" v-model.number="minimumTraffic" type="number" placeholder="2" value="2">
            

            <h3>En el topic</h3>


            <multiselect 
              v-model="selectedTopics" 
              :options="topicsNames"
              :multiple="true"
              >
            </multiselect>

        </div>
          <div class="column">
            <h4>Notificarme cada:</h4>
            <input class="input" type="number" placeholder="2" value="2">  
          </div>


          <div class="column">
            <div class="select">
              <select v-model="notificationsTimeMeasure">
                <option value="1" >Segundos</option>
                <option value="60" selected>Minutos</option>
                <option value="3600">Horas</option>
              </select>
            </div>
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
import echarts from 'echarts'
import { keys, sortBy, filter, values, mapValues, pick, map, zipObject, compact, includes, reduce } from 'lodash'

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
      messagesPerInterval: 1,
      secondsSpent: 0,
      selectedTime: 1,
      notificationsTime: 5,
      minimumTraffic: -50,
      notificationsTimeMeasure: 60,
      numeroSlice: -5,
      interval: 5000,
      selected: null,
      selectedTopics: [],
      newValues: []
    }
  },
  methods: {
    showNotification(titulo, message){

      this.$notification.show(titulo, {
      body: message
      }, {})

    },
    customLabel (option) {

      return `${option.library} - ${option.language}`
    },
    showNotificationInterval(){

      setInterval(function () {
          this.$notification.show('Trafico de mensajes', {
          body: 'Trafico de mensajes a llegado a ' + this.minimumTraffic
          }, {})
        }.bind(this), 5 * parseInt(60 * 1000))
    },
    createMultiplChart(){
    }
  },
  mounted () {

    Vue.use(VueNativeNotification, {
      requestOnNotify: true
    })


    setInterval(function () {
      this.secondsSpent++

      // It's always at the watch if it got into traffic minimum to show notification, every second
      if(this.minimumTraffic == this.messagesPerInterval){
        setInterval(function () {
          this.$notification.show('Trafico de mensajes', {
          body: 'Trafico de mensajes a llegado al minimo de ' + this.minimumTraffic
          }, {})
        }.bind(this), parseInt(this.notificationsTime) * parseInt(this.notificationsTimeMeasure) * 1000)//1 * parseInt(1 * 1000))
     }
    }.bind(this), 1000)


    // initialize echarts instance with prepared DOM
    const multipleChart = echarts.init(document.getElementById('main'))
    this.multipleChart = multipleChart
    /*NEW CHART*/
    multipleChart.setOption({
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
    ],
    series : [{
            type:'line',
            stack: '总量',
            areaStyle: {normal: {}}
        }
    ]
  })

    this.$nextTick(() => {
      window.addEventListener('resize', () => {
        [multipleChart].forEach(c => {
          c.resize()
        })
      })
    })


  },
  sockets: {
    exabeat (data) {

      this.counter++
      const now = new Date()
      const strDate = [now.getHours(), now.getMinutes()].join(':')

      this.data.push(data)
      this.date.push(strDate)
      this.data = this.data.slice(this.numeroSlice)
      this.date = this.date.slice(this.numeroSlice)
      this.interval = this.data[0].interval

      var tmp = []
      var dataTopics = this.data[0].topics

        for(let i in this.data[0].topics){

          const msg = this.data[0].topics[i] // message of each topic with its increment
          
          if(this.topics[msg.topicName]){ // if the array of topics has a name node of the topic, ie it exists
            tmp[msg.topicName] = Object.assign([], this.topics[msg.topicName])
            tmp[msg.topicName].push(msg.increment)
            tmp[msg.topicName] = tmp[msg.topicName].slice(this.numeroSlice)
            
          } else {
            let firstTimeIncrements = []
            for(s in 4){
              firstTimeIncrements.push(0)
            }
            firstTimeIncrements.push(msg.increment)

            tmp[msg.topicName] = [0,0,0,0, msg.increment]
          }
        }

        console.log(JSON.stringify(keys(tmp))+''+JSON.stringify(values(tmp)))
        this.topics = tmp
        this.topicsNames = keys(this.topics)

        
        this.newValues = values(mapValues(this.topics, (data, name) => {
          return {
            name: name, 
            type:'line',
            stack: '总量',
            areaStyle: {normal: {}},  
            data: data }
        }))

      
        this.multipleChart.setOption({
        legend: {
            data: this.topicsNames,
            type: 'scroll',
            left: 10
        },
        xAxis: {
          data: this.date
        },
         series : this.newValues
      })

      var newArray = this.newValues.reduce((results, value) => {
        const nombre = value.name
        return (this.selectedTopics.indexOf(nombre) > -1) ? [...results, { value, name }] : results;
      }, []);
      
      var valuesToCheck = newArray.map(value => value.value.data)
     // alert(JSON.stringify(newArray.map(value =>  value.value.data)))

      for(let a in valuesToCheck){
        if(valuesToCheck[a].includes(this.minimumTraffic)){
          this.showNotification('Topic ha llegado a '+ this.minimumTraffic,'Se ha llegado al minimo flujo de mensajes asignado: '+this.minimumTraffic)
        }
      }

    },
    exabeatTopicsChanged(data){

      if(data == 0){
        
        this.showNotification('Se han realizado cambios sobre el fichero de configuración', 'Archivo Config.json cambiado')

      }else{
        this.showNotification('Topics cambiados', 'Se han cambiado los topics desde el archivo de configuración')
      }
      
     }
  }
}
</script>

<style>
#app {
  margin-top: 30px
}
</style>





<style src="vue-multiselect/dist/vue-multiselect.min.css"></style>