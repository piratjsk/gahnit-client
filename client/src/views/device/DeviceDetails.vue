<template>
<div>
  <template v-if="$apollo.loading">
    <Loader/>
  </template>

  <template v-else>
    <router-link :to="{ name: 'DeviceEdit', params: { id: device.id }}"  tag="button">Edytuj</router-link>
    <button @click="deleteDevice(device.id)">Usun</button>
    <h1>{{ device.type.preferedName ? device.type.preferedName : device.type.name }} {{device.producent}} {{device.producentNumber}}{{device.udtNumber ? " - "+device.udtNumber : ""}}</h1>
    <ul>
      <li>Właściciel: <router-link :to="{ name: 'CustomerDetails', params: { id: device.owner.id }}">{{ device.owner.name }}</router-link></li>
      <li>Numer seryjny: {{device.serialNumber}}</li>
      <li>Numer UDT: {{device.UDTNumber}}</li>
      <li>Numer rejestracyjny: {{device.registrationNumber}}</li>
      <li>Rok produkcji: {{device.productionYear}}</li>
      <li>Producent: {{device.producent}}</li>
      <li>Uźwig: {{device.capacity}}</li>
      <li>Typ: {{device.producentNumber}}</li>
      <li>Rodzaj: <router-link :to="{ name: 'DeviceTypeDetails', params: { id: device.type.id }}">{{ device.type.name }}</router-link></li>
    </ul>
    <h4>Nadchodzące terminy</h4>
    <ul>
      <li v-for="term in getAproachingTerms()">
        {{ term.type }} - {{ term.nextDate }} ({{ term.exact }})
      </li>
    </ul>
    <h4>
      Wykonane akcje
      <router-link :to="{ name: 'ActionNew', params: { id: device.id, type: 'conservation' }}"  tag="button">Nowa konserwacja</router-link>
      <router-link :to="{ name: 'ActionNew', params: { id: device.id, type: 'udt' }}"  tag="button">Nowy odbiór UDT</router-link>
    </h4>
    <ul>
      <li v-for="action in device.actions" v-bind:key="action.id">
        {{getDate(action.date)}} - {{ getActionName(action) }} {{action.__typename=="UDT"?`(${action.inspector}) `:""}}<button @click="deleteAction(action.id)">Usuń</button>
      </li>
      <li v-if="device.actions.length == 0">Nic tu nie ma.</li>
    </ul>
    <div class="notesCont">
      <h3>Notatki</h3>
      <form @submit.prevent="saveNotes(device.id, notes)">
        <textarea id="notes" v-model="notes" rows="32"></textarea>
        <button>Zapisz</button>
      </form>
    </div>
  </template>
</div>
</template>
<style lang="stylus">
.notesCont
  position: absolute
  right: 20px
  top: 50px
  width: calc(100vw - 900px)
  
  textarea
    font-family: inherit
    border: none
    background-color: #c6c8ca
    padding: 5px
    border-radius: 2px
    font-size: inherit
</style>
<script>
  import Loader from '@/components/Loader'

  import moment from 'moment'
  import pl from 'moment/locale/pl'
  moment.locale('pl')

  import gql from 'graphql-tag'

  const DEVICE_QUERY = gql`
    query DeviceQuery($id: ID!) {
      device(id: $id) {
        id
        serialNumber
        UDTNumber
        registrationNumber
        productionYear
        producent
        producentNumber
        capacity
        nextUDT
        nextConservation
        notes
        type {
          id
          name
          preferedName
          conservationEveryNDays
          udtEveryNDays
        }
        owner {
          name
          id
        }
        actions {
          id
          date
          ...on UDT {
            inspector
          }
        }
      }
    }
  `
  const DELETE_MUTATION = gql`
    mutation deleteDevice($id: ID!) {
      deleteDevice(id: $id) {
        id
      }
    }
  `

  const DELETE_ACTION = gql`
    mutation removeAction($id: ID!) {
      removeAction(id: $id) {
        id
      }
    }
  `

  const UPDATE_NOTES = gql`
    mutation updateDevice($id: ID!, $data: DeviceUpdateInput!) {
      updateDevice(id: $id, data: $data) {
        id
      }
    }
  `

  export default {
    components: { Loader },
    data: () => ({
      device: {},
      notes: "",
    }),

    apollo: {
      device: {
        query: DEVICE_QUERY,
        variables() {
          return {id: this.$route.params.id}
        },
        update({device}) {
          const notes = JSON.parse(JSON.stringify(device.notes))
          this.notes = notes
          return device
        },
      },
    },

    methods: {
      saveNotes(id, notes) {
        this.$apollo.mutate({
          mutation: UPDATE_NOTES,
          variables: { id: id, data: {notes: notes} },
        }).then((data) => {
          alert("Notatki zostały zapisane.")
        }).catch((error) => {
          console.error(error)
          alert(error)
        })
      },
      getAproachingTerms() {
        const now = moment()
        const nextConservationDate = moment(this.device.nextConservation)
        const nextUDTDate = moment(this.device.nextUDT)

        const tillNextUDT = now.isSame(nextUDTDate, 'day') ? 'dzisiaj' : now.to(nextUDTDate)
        const tillNextConservation = now.isSame(nextConservationDate, 'day') ? 'dzisiaj' : now.to(nextConservationDate)

        return [
          { type: "Konserwacja", nextDate: tillNextConservation, exact: nextConservationDate.format("MM/YYYY") },
          { type: "Odbiór UDT", nextDate: tillNextUDT, exact: nextUDTDate.format("MM/YYYY") }
        ]

      },
      getDate(date) {
        return moment(new Date(date)).format("DD/M/YYYY")
      },
      getActionName(action) {
        if (action.__typename == "Conservation") return "Konserwacja"
        if (action.__typename == "UDT") return "Odbiór UDT"
        return "Serwis"
      },
      deleteDevice(id) {
        this.$apollo.mutate({
          mutation: DELETE_MUTATION,
          variables: { id },
          update: (proxy, { data: { deleteDevice } }) => {
            const query = gql`query { devices { id } }`
            const data = proxy.readQuery({query})
            const devices = data.devices.filter(device => { return device.id != deleteDevice.id})
            data.devices = devices
            proxy.writeQuery({ query, data })
          }
        }).then((data) => {
          this.$router.push({ path: '/devices' })
        }).catch((error) => {
          console.error(error)
          alert(error)
        })
      },
      deleteAction(id) {
        this.$apollo.mutate({
          mutation: DELETE_ACTION,
          variables: { id },
        }).then((data) => {
          this.$router.go()
        }).catch((error) => {
          console.error(error)
          alert(error)
        })
      }
    }
  }
</script>
