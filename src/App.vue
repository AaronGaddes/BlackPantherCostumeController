<template>
  <v-app>
    <v-app-bar
      app
      color="dark"
      dark
    >
      <div class="d-flex align-center">
        <v-icon
          alt="Vuetify Logo"
          class="shrink mr-2"
          transition="scale-transition"
          width="40"
        >mdi-cat</v-icon>

        <v-toolbar-title>{{title}}</v-toolbar-title>
      </div>

    </v-app-bar>
    <v-content>
      <v-container
          class="fill-height"
          fluid
        >
          <v-row
            align="center"
            justify="center"
          >
            <v-col
              cols="12"
              sm="8"
              md="4"
            >
              <v-banner
                  :single-line=true
                  :icon="connected ? 'mdi-bluetooth-connect' : 'mdi-bluetooth-off'"
                  :icon-color="connected ? 'teal lighten-2' : 'red'"
                  :elevation="1"
                >
                  <span v-if="connected">
                    Connected to <b>{{bluetoothDevice.name}}</b>
                  </span>
                  <span v-else>
                    Disconnected.
                  </span>
                  <template v-slot:actions v-if="!connected">
                    <v-btn
                      text
                      color="teal lighten-2"
                      @click="connectToBLE"
                      :disabled="connected"
                    >
                      Connect
                    </v-btn>
                  </template>
                </v-banner>
            </v-col>
          </v-row>
          <v-row
            align="center"
            justify="center"
          >
          <v-col
              cols="12"
              sm="8"
              md="4"
            >
            <v-card
              class="mx-auto"
              width="100%"
              :disabled="!connected"
            >
              <v-card-text>
                <v-row
                  justify="center"
                  >
                      <v-btn-toggle
                        v-model="pattern"
                        mandatory
                      >
                        <v-btn>
                          <v-icon>mdi-format-color-fill</v-icon>
                        </v-btn>
                        <v-btn>
                          <v-icon>mdi-string-lights</v-icon>
                        </v-btn>
                        <v-btn>
                          <v-icon>mdi-star-four-points-outline</v-icon>
                        </v-btn>
                        <v-btn>
                          <v-icon>mdi-ray-start</v-icon>
                        </v-btn>
                      </v-btn-toggle>
                  </v-row>
              </v-card-text>
            </v-card>
          </v-col>
          </v-row>
          <v-row
            align="center"
            justify="center"
          >
          <v-col
              cols="12"
              sm="8"
              md="4"
            >
            <v-card
                class="mx-auto"
                width="100%"
                :disabled="!connected"
              >
                <v-card-text>
                  <div class="d-flex justify-center">
                    <v-color-picker
                    :show-swatches=true
                    :hide-mode-switch=true
                    :disabled="!connected"
                    :mode.sync="colorPickerMode"
                    v-model="color"
                    ></v-color-picker>
                  </div>
                </v-card-text>
            </v-card>
          </v-col>
          </v-row>
        </v-container>
    </v-content>
  </v-app>
</template>

<script>

export default {
  name: 'App',

  components: {
  },

  data: () => ({
    title: 'Black Panther Costume Controller',
    bluetoothDevice: undefined,
    bluetoothDeviceServer: undefined,
    bluetoothDeviceService: undefined,
    bluetoothDeviceCharacteristics: undefined,
    bluetoothDeviceCharacteristicsByName: {},
    connected: false,
    selectedPattern: 0,
    baseColor: '#FF00FF',
    colorPickerMode: 'hexa'
  }),

  computed: {
      color: {
        get () {
          return this.baseColor
        },
        async set (v) {
          let colorChangedOnDevice = await this.sendColorToDevice(v);
          colorChangedOnDevice ? this.baseColor = v : this.baseColor = this.baseColor;
        },
      },
      pattern: {
        get () {
          return this.selectedPattern
        },
        async set (v) {
          let patternChangedOnDevice = await this.sendPatternToDevice(v);
          patternChangedOnDevice ? this.selectedPattern = v : this.selectedPattern = this.selectedPattern;
        },
      },
  },

  methods: {
    connectToBLE: async function(event) {
      event.preventDefault();
      console.log('Requesting Bluetooth Device...');
      try {
        // console.log('with ' + JSON.stringify(options));
        this.bluetoothDevice = await navigator.bluetooth.requestDevice({acceptAllDevices: true, optionalServices: [0x1700]});
        if(this.bluetoothDevice) {
          this.bluetoothDevice.addEventListener('gattserverdisconnected', this.onDisconnected);
        }
        console.log('device', this.bluetoothDevice);
      } catch(error)  {
        console.log('Argh! ' + error);
      }

      try {
        console.log('Connecting to GATT Server...');
        this.bluetoothDeviceServer = await this.bluetoothDevice.gatt.connect();
      } catch(error)  {
        console.log('Argh! ' + error);
      }

      try {
        console.log('Getting Service...');
        this.bluetoothDeviceService = await this.bluetoothDeviceServer.getPrimaryService(0x1700);
      } catch(error)  {
        console.log('Argh! ' + error);
      }

      try {

        console.log('Getting Characteristics...');

        this.bluetoothDeviceCharacteristics = await this.bluetoothDeviceService.getCharacteristics();

        console.log('> Characteristics: ',
          this.bluetoothDeviceCharacteristics);

          await this.bluetoothDeviceCharacteristics.forEach(async c => {
            let descriptor = await c.getDescriptor('gatt.characteristic_user_description')
            let value = await descriptor.readValue();

            let decoder = new TextDecoder('utf-8');

            let name = decoder.decode(value)
            console.log('name:', name);

            let characterValue = await c.readValue();

            switch (name) {
              case 'Color':
                this.baseColor = `#${decoder.decode(characterValue)}`;
                console.log('color from device:', this.baseColor);
                
                break;
              case 'Pattern':
                this.selectedPattern = Number.parseInt(decoder.decode(characterValue));
                console.log('pattern changed from device:', this.selectedPattern);
                
                break;
              default:
                break;
            }

            this.bluetoothDeviceCharacteristicsByName[name] = c;
          });

        
        console.log('> bluetoothDeviceCharacteristicsByName: ',
          this.bluetoothDeviceCharacteristicsByName);

        console.log('> Name:             ' + this.bluetoothDevice.name);
        console.log('> Id:               ' + this.bluetoothDevice.id);
        console.log('> Connected:        ' + this.bluetoothDevice.gatt.connected);

        this.connected = true;

      } catch(error)  {
        console.log('Argh! ' + error);
        this.connected = false;
      }
    },
    sendColorToDevice: async function (color){
      let success = false;
      let encoder = new TextEncoder('utf-8');
      color = encoder.encode(color.slice(1));
      try {
        await this.bluetoothDeviceCharacteristicsByName['Color'].writeValue(color);
        console.log(`color changed to ${color}`);
        success = true;
      } catch(error)  {
        console.log('Argh! ' + error);
        success = false;
      }
      return success;
    },
    sendPatternToDevice: async function(patternIndex) {
      let success = false;
      let encoder = new TextEncoder('utf-8');
      let pattern = encoder.encode(patternIndex);
      try {
        await this.bluetoothDeviceCharacteristicsByName['Pattern'].writeValue(pattern);
        console.log(`pattern changed to ${pattern}`);
        success = true;
      } catch(error)  {
        console.log('Argh! ' + error);
        success = false;
      }
      return success;
    },
    onDisconnected () {
      // Object event.target is Bluetooth Device getting disconnected.
      // console.log('Bluetooth Device disconnected');
      this.connected = false;
    }
  }
};
</script>
