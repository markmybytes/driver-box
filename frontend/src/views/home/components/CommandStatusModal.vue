<script setup lang="ts">
import CrossIcon from '@/components/icons/CrossIcon.vue'
import ModalFrame from '@/components/modals/ModalFrame.vue'
import * as executor from '@/wailsjs/go/execute/CommandExecutor'
import * as runtime from '@/wailsjs/runtime/runtime'
import AsyncLock from 'async-lock'
import { ref, useTemplateRef } from 'vue'
import { useI18n } from 'vue-i18n'
import { useToast } from 'vue-toast-notification'
import type { Command, Process } from '../types'
import TaskStatus from './TaskStatus.vue'

const frame = useTemplateRef('frame')

defineExpose({
  show: async (parallel: boolean, cmds: Array<Command>) => {
    frame.value?.show()

    isParallel = parallel

    processes.value = cmds.map(vals => ({ command: { ...vals }, status: 'pending' }))
    dispatchCommand()
  },
  hide: frame.value?.hide || (() => {})
})

const emit = defineEmits<{ completed: [] }>()

const { t } = useI18n()

const $toast = useToast({ position: 'top-left', duration: 7000 })

const lock = new AsyncLock()

let isParallel = false

const processes = ref<Array<Process>>([])

runtime.EventsOn('execute:exited', async (id: string, result: NonNullable<Process['result']>) => {
  const process = processes.value.find(c => c.procId === id)!
  process.result = result

  if (result.aborted) {
    process.status = 'aborted'
  } else if (![0, ...process.command.config.allowRtCodes].includes(result.exitCode)) {
    process.status = 'failed'
  } else if (result.lapse < process.command.config.minExeTime) {
    process.status = 'speeded'
  } else {
    process.status = 'completed'
  }

  dispatchCommand().then(() => {
    if (processes.value.every(c => c.status === 'completed')) {
      emit('completed')
      $toast.success(t('toast.finished'), { position: 'bottom-right' })
    } else if (processes.value.every(c => !c.status.includes('ing'))) {
      $toast.info(t('toast.finished'), { position: 'bottom-right' })
    }
  })
})

function getProcessName(process: Process) {
  return process.command.name
    ? `${process.command.groupName} - ${process.command.name}`
    : process.command.groupName
}

async function dispatchCommand() {
  lock.acquire('executor', async () => {
    const pendings = processes.value
      .filter(c => c.status === 'pending')
      .slice(0, isParallel ? undefined : 1)

    for (const process of pendings) {
      if (
        !process.command.config.incompatibles.every(id =>
          processes.value.filter(p => p.status === 'running').every(p => p.command.id != id)
        )
      ) {
        continue
      }

      await executor
        .Run(process.command.config.program, process.command.config.options)
        .then(processId => {
          process.status = 'running'
          process.procId = processId
        })
        .catch(error => {
          process.status = 'broken'
          process.result = {
            lapse: -1,
            exitCode: -1,
            stdout: '',
            stderr: '',
            error: (error as Error).toString(),
            aborted: false
          }
        })
    }
  })
}

function handleAbort(process: Process) {
  return lock
    .acquire('executor', () => {
      if (process.status == 'pending' || process.status == 'running') {
        process.status =
          process.procId == undefined || process.procId == '' ? 'aborted' : 'aborting'
      }
    })
    .then(() => {
      if (process.status != 'aborting') {
        return
      }

      // `aborted` status will be updated at `execute:exited` event handler
      executor.Abort(process.procId!).catch(error => {
        if (error.includes('process does not exist')) {
          $toast.warning(
            t('toast.cancelCompletedFailed', {
              name: getProcessName(process)
            })
          )
          return
        }

        error
          .toString()
          .split('\n')
          .forEach((err: string) => {
            if (err.includes('abort failed')) {
              $toast.warning(
                t('toast.cancelFailed', {
                  name: getProcessName(process)
                })
              )
            } else {
              $toast.error(`[${getProcessName(process)}] ${err}`)
            }
          })

        process.status = 'broken'
        process.result = {
          lapse: -1,
          exitCode: -1,
          stdout: '',
          stderr: '',
          error: error.toString(),
          aborted: false
        }
      })
    })
}
</script>

<template>
  <ModalFrame :on-demand="true" :immediate="false" ref="frame">
    <div class="w-[65vw] max-w-3xl">
      <!-- Modal content -->
      <div class="bg-white rounded-sm shadow-sm">
        <!-- Modal header -->
        <div class="flex items-center justify-between px-3 py-1.5 border-b rounded-t">
          <h3 class="font-semibold">
            {{ $t('executeStatus.title') }}
          </h3>
          <button
            type="button"
            class="inline-flex justify-center items-center h-8 w-8 ms-auto text-sm text-gray-400 enabled:hover:text-gray-900 bg-transparent enabled:hover:bg-gray-200 rounded-lg"
            @click="frame?.hide()"
            :disabled="
              processes.some(cmd => ['pending', 'running', 'aborting'].includes(cmd.status))
            "
          >
            <CrossIcon></CrossIcon>
          </button>
        </div>

        <!-- Modal body -->
        <div class="max-h-[70vh] overflow-y-auto py-2 px-4">
          <template v-for="(process, i) in processes" :key="i">
            <TaskStatus :process="process" @abort="handleAbort(process)"></TaskStatus>
          </template>
        </div>

        <div
          class="flex justify-end pb-2 px-4"
          v-show="
            processes.every(p => p.status.includes('ed')) &&
            processes.some(p => p.status != 'completed')
          "
        >
          <button
            class="flex rounded px-1 text-white bg-half-baked-600"
            @click="
              event => {
                $emit('completed')
                $toast.success(t('toast.finished'), { position: 'bottom-right' })

                // @ts-ignore
                event.currentTarget?.remove()
              }
            "
          >
            {{ $t('executeStatus.forceComplete') }}
          </button>
        </div>
      </div>
    </div>
  </ModalFrame>
</template>
