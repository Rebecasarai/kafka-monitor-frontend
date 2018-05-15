# kafka-monitor-frontend

> Frontend for the Kafka Monitor


The main part of the monitor app is at "App.vue", this is where  vue app is initialized, containing:
* The template that renders as html using the Vue sintax
* Javascript script where the vue app opions are setted and the lifecycle is followed. All the methods, imports and logic is done here. Important functions are for example the socket ones, where it receives a message from the server to visualize the data on a chart, every certain time.


## Build Setup

``` bash
# install dependencies
npm install

# serve with hot reload at localhost:8080
npm run dev

# build for production with minification
npm run build

# build for production and view the bundle analyzer report
npm run build --report
```



For a detailed explanation on how things work, check out the [guide](http://vuejs-templates.github.io/webpack/) and [docs for vue-loader](http://vuejs.github.io/vue-loader).
