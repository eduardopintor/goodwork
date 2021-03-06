<template>
<div :class="{'hidden': (activeTab != 'messages')}" class="w-full">
    <div class="flex flex-col bg-white mx-auto my-8 max-w-md shadow rounded">

        <div class="text-grey-dark bg-white shadow p-4 text-xl flex flex-row items-center">
            <div>
                Currently in room:
            </div>
            <template v-for="user in users">
              <img :src="generateUrl(user.avatar)" alt="" class="w-8 h-8 rounded-full mr-2 ml-4">
            </template>
        </div>

        <div id="message-box" class="h-50-vh overflow-y-auto">
            <div class="">
                <div v-for="message in messages" class="flex flex-col p-4">
                    <div v-if="showDate(message.created_at)" class="w-full flex flex-row py-4">
                        <div class="border-b w-1/5 flex-grow mb-2"></div>
                        <div class="text-grey-dark text-sm text-center px-4">
                            {{ getDate(message.created_at) }}
                        </div>
                        <div class="border-b w-1/5 flex-grow mb-2"></div>
                    </div>
                    <div class="flex flex-row text-grey-darker py-3"
                        :class="[(message.user_id == user.id) ? 'self-end' : '']">
                        <img :src="generateUrl(message.user.avatar)" :alt="message.user.name" class="w-10 h-10 rounded-full border-2"
                            :class="[(message.user_id == user.id) ? 'order-1 border-teal' : 'border-pink']">
                        <div class="rounded-2xl p-4 mx-2 w-64 leading-normal mt-6 text-grey-darkest"
                            :class="[(message.user_id == user.id) ? 'bg-teal-lightest rounded-tr-none' : 'bg-pink-lightest rounded-tl-none']">
                            <div>{{ message.body }}</div>
                            <div class="text-grey-darkest text-xs pt-2 float-right">{{ getTime(message.created_at) }}</div>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <div class="relative bg-grey-light">
            <div class="static text-center p-8">
                <textarea class="static textarea resize-none rounded w-full p-4 text-grey-darker"
                        id="send-message"
                        placeholder="write your message here"
                        rows=1
                        v-model="message"
                        @keydown.enter.prevent="sendMessage($event)"
                        @focus="clearTitleNotification()"></textarea>
            </div>
        </div>
    </div>
</div>
</template>

<script>
import luxon from 'luxon'
let day = null
export default {
  props: ['resource', 'resourceType', 'activeTab'],
  data: () => ({
    messages: [],
    message: '',
    user: navbar.user,
    unreadMessage: 0,
    title: '',
    users: []
  }),
  mounted () {
    axios.get('/messages', {
      params: {
        resource_type: this.resourceType,
        resource_id: this.resource.id
      }
    })
      .then((response) => {
        this.messages = response.data.messages.reverse()
      })
      .catch((error) => {
        console.log(error)
      })
    this.title = document.title
    this.listen()
    document.getElementById('message-box').scrollTop = document.getElementById('message-box').scrollHeight
  },
  updated () {
    document.getElementById('message-box').scrollTop = document.getElementById('message-box').scrollHeight
  },
  methods: {
    sendMessage (e) {
      if (e.shiftKey) {
        this.message = this.message + '\n'
      } else {
        var msg = this.message
        this.message = ''
        axios.post('/messages', {
          message: msg,
          resource_type: this.resourceType,
          resource_id: this.resource.id
        })
          .then((response) => {
            if (response.data.status == 'success') {
              response.data.message.user = navbar.user
              this.messages.push(response.data.message)
            }
          })
          .catch((error) => {
            console.log(error)
          })
      }
    },
    listen () {
      Echo.join(this.resourceType + '.' + this.resource.id)
          .here(users => {
            this.users = users
          })
          .joining(user => {
            this.users.push(user)
          })
          .leaving(user => {
            this.users = this.users.filter(u => u.username !== user.username)
          })
          .listen('MessageCreated', event => {
            event.message.user = event.user
            this.messages.push(event.message)
            if ((document.activeElement != document.getElementById('send-message')) || (!document.hasFocus())) {
              this.unreadMessage += 1
              document.title = '(' + this.unreadMessage + ') ' + this.title
            }
          })
    },
    clearTitleNotification () {
      document.title = this.title
      this.unreadMessage = 0
    },
    showDate (created_at) {
      let createdDay = luxon.DateTime.fromSQL(created_at).toLocaleString(luxon.DateTime.DATE_MED)
      if (day === null || day != createdDay) {
        day = createdDay
        return true
      }
      return false
    },
    getDate (created_at) {
      return luxon.DateTime.fromSQL(created_at).toLocaleString(luxon.DateTime.DATE_MED)
    },
    getTime (created_at) {
      return luxon.DateTime.fromSQL(created_at).toLocaleString(luxon.DateTime.TIME_SIMPLE)
    }
  }
}
</script>