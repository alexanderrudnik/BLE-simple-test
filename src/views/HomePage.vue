<template>
  <Suspense>
    <template #default>
      <ion-page>
        <ion-header :translucent="true">
          <ion-toolbar>
            <ion-title>Blank</ion-title>
          </ion-toolbar>
        </ion-header>

        <ion-content :fullscreen="true">
          <ion-header collapse="condense">
            <ion-toolbar>
              <ion-title size="large">Blank</ion-title>
            </ion-toolbar>
          </ion-header>
          <pre class="ion-margin-bottom">{{logs}}</pre>
          <ion-button @click="logs = []">Clear</ion-button>
          <template v-if="!connectedDevice">
            <ion-button @click="scan">Scan</ion-button>
            <ion-list>
              <ion-item v-for="(device, index) in devices" :key="index" button @click="connect(device)">{{device.name}}</ion-item>
            </ion-list>
          </template>
          <template v-else>
            <ion-button @click="disconnect">Disconnect</ion-button>
            <ion-button @click="write">Write</ion-button>
          </template>

        </ion-content>
      </ion-page>
    </template>
    <template #fallback>LOADING</template>
  </Suspense>

</template>

<script lang="ts">
import { IonContent, IonHeader, IonPage, IonTitle, IonToolbar } from '@ionic/vue';
import {BleClient, BleDevice, dataViewToText, ScanResult, textToDataView} from '@capacitor-community/bluetooth-le';
import {defineComponent, onBeforeUnmount, ref} from "vue";

export const SERVICE = '331a36f5-2459-45ea-9d95-6142f0c4b307'
export const TX_CHAR = 'a73e9a10-628f-4494-a099-12efaf72258f' // This characteristic is used for receiving characters from the device
export const RX_CHAR = 'a9da6040-0823-4995-94ec-9ce41ca28833' // This characteristic is used for transmitting things from our mobile to the device


export default defineComponent({
  name: 'HomePage',
  components: {IonContent, IonHeader, IonPage, IonTitle, IonToolbar},
  setup() {
    const logs = ref<string[]>([])

    BleClient.initialize({androidNeverForLocation: true}).then(() => {
      log('Init success')
      BleClient.isLocationEnabled().then(res => log(`Location: ${res}`))
    });

    const devices = ref<BleDevice[]>([])
    const connectedDevice = ref<BleDevice>()

    const scan = async () => {
      if (connectedDevice.value) {
        await BleClient.disconnect(connectedDevice.value?.deviceId)
      }

      devices.value = []
      await BleClient.requestLEScan({services: [SERVICE]}, (res: ScanResult) => {
        devices.value = devices.value.concat(res.device)
    })

      setTimeout(() => {
        BleClient.stopLEScan().then(() => log('Stop scanning'));
      }, 5000)
    }

    const connect = async (device: BleDevice) => {
      log('Connecting to ' + device.name)
      await BleClient.disconnect(device.deviceId)
      BleClient.connect(device.deviceId).then(() => {
        log('Connected successfully')
        BleClient.createBond(device.deviceId).then(() => {
          connectedDevice.value = device
          log('Bonded successfully')
        })


        BleClient.startNotifications(device.deviceId, SERVICE, TX_CHAR, result => log(`RESPONSE: ${_parseData(result)}`))
            .then(() => log('Notifications started'))
      })
    }

    const log = (str: string) => {
      logs.value = logs.value.concat(str)
    }

    onBeforeUnmount(() => {
      disconnect()
    })

    const disconnect = () => {
      if (connectedDevice.value) {
        BleClient.stopNotifications(connectedDevice.value.deviceId, SERVICE, TX_CHAR).then(() => {
          log('Stop Notifications')

          if (connectedDevice.value) {
            BleClient.disconnect(connectedDevice.value.deviceId).then(() => {
              connectedDevice.value = undefined
              log('Disconnected success')
            })
          }
        })

      }
    }

    const write = () => {
      if (connectedDevice.value) {
        const cmd = `{"cmd":1}\n`

        BleClient.write(connectedDevice.value.deviceId, SERVICE, RX_CHAR, _parseString(cmd)).then(() => log(`${cmd} written`))
      }

    }

    const _parseData = (data: any) => {
      return dataViewToText(data)
    }

    const _parseString = (string: string) => {
      return textToDataView(string)
    }

    return {logs, scan, connect, devices, connectedDevice, disconnect, write}
  }
})



</script>

<style scoped>
</style>
