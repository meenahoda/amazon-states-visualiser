<template>
  <q-page class="bg-dark text-light">
    <q-layout-header>
      <q-toolbar color="dark">
        <q-toolbar-title>
          Amazon state diagram visualiser
        </q-toolbar-title>
        <q-btn-dropdown label="Examples" class="q-mr-sm" outline text-color="white">
          <q-list link>
            <q-item
              v-for="(sm, key) in stateMachines"
              :key="key"
              v-close-overlay
              @click.native="selectStateMachine(key)"
              v-if="key !== 'help'"
            >
              <q-item-main>
                <q-item-tile label>{{sm.Comment}}</q-item-tile>
              </q-item-main>
            </q-item>
          </q-list>
        </q-btn-dropdown>
      </q-toolbar>
    </q-layout-header>
    <div class="row">
      <div class="col-6 shadow-6">
        <brace
          fontsize="12px"
          theme="clouds_midnight"
          mode="json"
          codefolding="markbegin"
          softwrap="free"
          selectionstyle="text"
          highlightline
          style="height: calc(100vh - 50px); width: 100%"
          @code-change="codeChange"
        />
      </div>
      <div class="col-6">
        <inspect-modal class="col-6" :opened="modalOpen" :data="modalData" :id="modalId" @close="modalOpen=false"/>
        <div class="col-6 q-pa-xl" v-if="reset">
          <p class="q-display-3 text-weight-thin">Select a state machine example to visualise!</p>
        </div>
        <div class="col-6 q-pa-xl scroll" v-if="!reset" style="max-height: calc(100vh - 50px);">
          <q-card class="bg-faded q-mb-md">
            <q-card-title>Start at: {{JSON.parse(stateCode).StartAt}}</q-card-title>
          </q-card>
          <q-card
            v-for="(value, key) in flattenedStates"
            :key="key"
            class="bg-tertiary q-mb-md"
          >
            <q-card-title>
              <div class="row">
                <div class="col">{{key}}</div>
                <div class="col text-right">
                  <q-chip color="faded" dense square>{{value.Type}}</q-chip>
                </div>
              </div>
            </q-card-title>
            <q-card-main>
              <div v-if="value.Type === 'Choice'">
                <div
                  v-for="(choice, idx) in value.Choices"
                  :key="idx"
                >
                  <div>{{formatChoice(choice)}}</div>
                </div>
              </div>

              <div v-if="value.Type === 'Parallel'">
                <div
                  v-for="(branch, idx) in value.Branches"
                  :key="idx"
                >
                  Branch {{idx + 1}}
                  Start at: {{branch.StartAt}}
                </div>
              </div>

              <div v-if="value.Next">Go to: {{value.Next}}</div>
              <div v-if="value.End">End</div>
            </q-card-main>
          </q-card>
        </div>
        <p class="q-caption text-light fixed-bottom-right q-pr-md">
          Inspired by the
          <a class="text-info" href="https://github.com/wmfs/tymly-monorepo">Tymly</a>
          implementation of
          <a class="text-info"
             href="https://docs.aws.amazon.com/step-functions/latest/dg/concepts-amazon-states-language.html">Amazon
            States Language</a>
        </p>
      </div>
    </div>

    <div style="position: fixed; bottom: 18px; right: 18px; text-align: right;">
      <q-btn round color="info" outline icon="clear" size="md" style="bottom: 20px; margin-right: 10px;" @click="clear">
        <q-tooltip>Clear</q-tooltip>
      </q-btn>
      <q-btn round color="info" icon="refresh" size="md" style="bottom: 20px" @click="refresh">
        <q-tooltip>Refresh</q-tooltip>
      </q-btn>
    </div>
  </q-page>
</template>

<style>
  ::-webkit-scrollbar {
    width: 10px;
  }

  ::-webkit-scrollbar-track {
    background: #272822;
  }

  ::-webkit-scrollbar-thumb {
    background: #888;
  }

  ::-webkit-scrollbar-thumb:hover {
    background: #555;
  }

  rect {
    cursor: pointer;
  }
</style>

<script>
import * as stateMachines from '../state-machines'
import Brace from 'vue-bulma-brace'
import * as brace from 'brace'
import InspectModal from 'components/StateInspector.vue'

export default {
  name: 'PageIndex',
  components: {
    Brace,
    InspectModal
  },
  data: function () {
    return {
      stateCode: '',
      flattenedStates: {},
      stateMachines,
      modalOpen: false,
      modalData: {},
      modalId: '',
      reset: true
    }
  },
  methods: {
    selectStateMachine (key) {
      this.stateCode = JSON.stringify(this.stateMachines[key], null, 2)
      this.refreshEditor()
      this.refresh()
    },
    toggleModal (id) {
      this.modalOpen = true
      this.modalId = id
      this.modalData = JSON.parse(this.stateCode).States[id]
    },
    codeChange (e) {
      this.stateCode = e
    },
    refresh () {
      try {
        this.reset = false
        this.flattenStates(this.stateCode)
      } catch (e) {
        console.log('Error:', e)
        this.$q.notify({
          type: 'warning',
          message: 'There was an error.',
          position: 'top'
        })
      }
    },
    clear () {
      this.stateCode = ''
      this.refreshEditor()
      this.reset = true
    },
    refreshEditor () {
      this.editor.session.setValue(this.stateCode)
    },
    flattenStates (stateMachine) {
      this.flattenedStates = {}
      const flatten = sm => {
        if (sm.States) {
          Object.entries(sm.States).forEach(([key, value]) => {
            if (!this.flattenedStates[key]) this.flattenedStates[key] = value

            if (value.Branches) {
              value.Branches.forEach(branch => {
                flatten(branch, this.flattenedStates)
              })
            }
          })
        }
      }

      flatten(JSON.parse(stateMachine))
    },
    formatChoice (choice) {
      if (choice.StringEquals) {
        return `If ${choice.Variable} is equal to ${choice.StringEquals} then go to: ${choice.Next}`
      }
      // todo: other types of choices
    }
  },
  mounted () {
    this.editor = brace.edit('vue-bulma-editor')
    this.stateCode = JSON.stringify(stateMachines['help'], null, 2)
    this.refreshEditor()
  }
}
</script>
