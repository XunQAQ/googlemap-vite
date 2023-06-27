<template>
  <div>
    <div class="operation-wrap">
      <div>
        <v-btn class="text-subtitle-1" color="#409eff" rounded="2" @click="getCurrentLocation" :loading="loading">
          Get Current location
        </v-btn>
      </div>
      <div class="search">
        <v-text-field clearable v-model="searchQuery" label="Please enter search content"
                      @keyup.enter="searchLocation"></v-text-field>
        <v-btn class="text-subtitle-1 search-btn" color="#409eff" rounded="2" size="large" @click="searchLocation">
          Search
        </v-btn>
      </div>
    </div>

    <div id="map"></div>
    <div class="table-wrap">
      <v-btn class="text-subtitle-1" color="red" rounded="2" @click="deleteSelectedLocations">
        delete
      </v-btn>
      <v-table>
        <thead>
        <tr>
          <th></th>
          <th class="text-left">Location</th>
          <th class="text-left">TimeZone</th>
          <th class="text-left">Time</th>
        </tr>
        </thead>
        <tbody>
        <tr v-for="location in pageData" :key="location.id">
          <td><input type="checkbox" v-model="selectedLocations" :value="location.id"></td>
          <td>{{ location.name }}</td>
          <td>{{ location.timezone }}</td>
          <td>{{ location.time }}</td>
        </tr>
        </tbody>
      </v-table>
      <v-pagination v-model="pageNum" :length="pagerCount(locations.length, pageSize)"></v-pagination>
    </div>
  </div>
</template>




<script>

import axios from 'axios';
import {toRaw} from "vue";

export default {
  name: 'LocationSearch',
  data() {
    return {
      pageNum: 1,
      pageSize: 10,
      loading: false,
      searchQuery: '',
      locations: [],
      pageData: [],
      selectedLocations: [],
      map: null
    };
  },
  watch: {
    pageNum(num) {
      let size = this.pageSize
      this.pageData = this.locations.slice(num * size - size, num * size)
    }
  },
  mounted() {
    /*
    * initializing the map
    * */
    this.initMap();
  },
  methods: {
    /*
    * Get current position
    * */
    getCurrentLocation() {
      this.loading = true
      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition(this.handleCurrentLocation);
      }
    },
    /*
    * Add marker to current position
    * */
    handleCurrentLocation(position) {
      const {latitude, longitude} = position.coords;
      this.updateMap(latitude, longitude, () => {
      });
    },
    /*
    * Google map search api
    * */
    searchLocation() {
      if (this.searchQuery) {
        axios.get(`https://maps.googleapis.com/maps/api/geocode/json?address=${this.searchQuery}&key=AIzaSyAtCgHsNw_NrVT4mMQ1oAA9b8C0UaFMOz8`)
          .then(response => {
            const {results} = response.data;
            if (results && results.length > 0) {
              const {lat, lng} = results[0].geometry.location;
              // Update the map and carry marker data
              this.updateMap(lat, lng, marker => {
                this.getTimezone(lat, lng, results, marker)
              });
            }
          })
          .catch(error => {
            console.error(error);
          });
      }
    },
    /*
    * Map initialization
    * */
     initMap() {
      this.map = new google.maps.Map(document.getElementById('map'), {
        center: {lat: 0, lng: 0},
        zoom: 12
      });
    },
    /*
    * Update, add map markers
    * */
    updateMap(latitude, longitude, callback) {
      const latLng = new google.maps.LatLng(latitude, longitude);
      this.map.setCenter(latLng);
      const marker = new google.maps.Marker({
        position: latLng,
        map: this.map
      });
      callback({
        marker: marker
      })
      this.loading = false
    },
    // Get the time zone using timezonedb
    getTimezone(latitude, longitude, results, marker) {
      const url = `https://api.timezonedb.com/v2.1/get-time-zone?key=QJMU3IFQVISK&format=json&by=position&lat=${latitude}&lng=${longitude}`;
      // Send request to Time Zone API
      fetch(url)
        .then(response => response.json())
        .then(data => {
          this.locations.unshift({
            id: results[0].place_id,
            name: results[0].formatted_address,
            timezone: data.zoneName,
            time: data.formatted,
            marker: marker.marker
          });
          let size = this.pageSize
          this.pageData = this.locations.slice(this.pageNum * size - size, this.pageNum * size)
        })
        .catch(error => {
          this.locations.unshift({
            id: results[0].place_id,
            name: results[0].formatted_address,
            timezone: 'Failed to get time zone',
            time: 'Failed to get time',
            marker: marker.marker
          });
          console.log('Failed to get time zone', error);
        });
    },
    deleteSelectedLocations() {
      // Remove markers from the map for deleted locations
      this.locations.forEach(item => {
        this.selectedLocations.forEach(selected => {
          if (item.id === selected) {
            toRaw(item.marker).setMap(null)
          }
        })
      })
      // Remove the checked data from the search data
      this.locations = this.locations.filter(location => !this.selectedLocations.includes(location.id));
      let size = this.pageSize
      let pageNum = this.pagerCount(this.locations.length, size)
      this.pageData = this.locations.slice(pageNum * size - size, pageNum * size)
    },
    /*
    * Calculate paging
    * */
    pagerCount(count, pageSize) {
      if (typeof (count) == "number") {
        if (count > 0) {
          try {
            let _pagerCount = count % pageSize == 0 ? count / pageSize : count / pageSize + 1;
            let c = _pagerCount.toFixed(0);//Decimal rounding
            _pagerCount = c > _pagerCount ? c - 1 : c;//Filter rounding
            // console.log(_pagerCount);
            // console.log(this.locations);
            // this.pageData = this.locations.slice(_pagerCount*2 - 2, _pagerCount*2)
            return _pagerCount;
          } catch (error) {
            return 0;
          }
        } else {
          return 0;
        }
      } else {
        return 0;
      }
    }
  }
};
</script>

<style>
#map {
  width: 100%;
  height: 400px;
}

.operation-wrap {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
  box-sizing: border-box;
  padding: 20px;
}

.text-subtitle-1 {
  margin-right: 20px;
  color: #ffffff !important;
}

.search {
  width: 300px;
  margin-left: 20px;
  display: flex;
  flex-direction: row;
}

.search-btn {
  margin-top: 6px;
  margin-left: 10px;
}

.table-wrap {
  box-sizing: border-box;
  padding: 20px;

}
</style>
