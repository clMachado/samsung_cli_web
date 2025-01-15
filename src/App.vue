
<!-- para rodar, acessa pasta do projeto e dar o comando     npm run serve   -->
<template>
  <div id="app">
    <h1>Controle do Ar-Condicionado Samsung</h1>

    <div>
      <input 
        v-model="pwd" 
        type="text" 
        placeholder="Informe a Senha" 
      />
      <button @click="fetchDevices">Carregar</button>
      <!-- Exibição do resultado -->
      <div v-if="result">
        <h3><p>Resultado: {{ result }}</p></h3>
      </div>

      
    </div>

    <div v-if="logon">
        <div v-if="loading">
          <p>Carregando dispositivos...</p>
        </div>

        <div v-else>
          <ul>
            <li v-for="device in devices" :key="device.deviceId">
              <h2>{{ device.label || device.name }}</h2>
              <p>ID: {{ device.deviceId }}</p>
              <p>Capacidades: {{ device.capabilities.join(", ") }}</p>

              <!-- Botões para comandos -->
              <button @click="sendCommand(device.deviceId, 'on', 'switch')">Ligar</button>
              <button @click="sendCommand(device.deviceId, 'off', 'switch')">Desligar</button>
              <!--button @click="sendCommand(device.deviceId, 'setCoolingSetpoint', 'thermostatCoolingSetpoint', [22])">
              Ajustar para 22°C
              </button-->


              <button @click="sendModoEco(device.deviceId)">
              Modo Eco
              </button>


              <button @click="getDeviceConfig(device.deviceId, 'thermostatCoolingSetpoint')">Consulta CFGs</button>

            </li>
          </ul>
        </div>
      
        <!-- Seção de feedback -->
      <div  id="resp">
        <div v-if="commandResponse">
          <h2>Resposta da API:</h2>
          <pre>{{ commandResponse }}</pre>
        </div>
      
        <div v-if=deviceConfig>
          <template>
            <div>
              <h2>Configurações do Ar-Condicionado</h2>
              <pre>{{ deviceConfig }}</pre>
            </div>
          </template>
        </div> 

      </div>
    </div>
</div>
 
</template>

<script>
import axios from "axios";

export default {
  data() {
    return {
      devices: [],
      loading: true,
      logon: false,
      pwd:'',
      result:'',
      commandResponse: null, // Para armazenar a resposta da API
      deviceConfig: {}, // configuracoes do dispositivo
    };
  },
  
  
  methods: {
    sleep(ms) {
      return new Promise((resolve) => setTimeout(resolve, ms));
    },
 
    // CARREGA OS DISPOSITIVOS CADASTRADOS
    async fetchDevices() {
      var response;
      response = "Falha";
      try {
        this.result = '';
        this.logon=false;

        response = await axios.get("http://localhost:3001/devices",{ headers: {pwd:this.pwd.trim()}, });
        // Extrair informações dos dispositivos
        this.devices = response.data.items.map((device) => ({
          deviceId: device.deviceId,
          name: device.name,
          label: device.label,
          capabilities: device.components[0].capabilities.map((cap) => cap.id),
        }));
        this.loading = false;
        this.logon=true;
      } catch (error) {
        console.error("Erro ao carregar dispositivos:", error);
        this.result = error.response.data.error
        this.loading = false;
      }
    },

    // ENVIA COMANDOS PARA O DISPOSITIVOS
    async sendCommand(deviceId, command, capability, args = []) {
      try {
        this.commandResponse = ''
        const response = await axios.post(`http://localhost:3001/device/${deviceId}/command`, {          
          command,
          capability,
          arguments: args,
        }, 
        {
          headers: {pwd:this.pwd.trim()},
        }
      );
        // Exibir a resposta no console ou na interface
        console.log("Resposta da API:", response.data);

        // Armazena a resposta na variável commandResponse
        this.commandResponse = JSON.stringify(response.data, null, 2);
        //alert(`Comando ${command} enviado com sucesso!`);

      } catch (error) {
        console.error(`Erro ao enviar comando ${command}:`, error.response?.data || error.message);
        this.commandResponse = `Erro: ${error.response?.data?.error || error.message}`;
      }
    },

    // CARREGA INDORMACOES DO DISPOSITIVO
    async getDeviceConfig(deviceId, command) {
      console.log(deviceId);
      console.log(command);

      const url = `http://localhost:3001/device/${deviceId}/get/${command}`;
      console.log(url)

      try {
        const response = await axios.get(url,{headers: {pwd:this.pwd.trim()},});

        this.deviceConfig = response.data.data || {};
        console.log("Configurações do Dispositivo:", this.deviceConfig);
      } catch (error) {
        console.error("Erro ao recuperar configurações do dispositivo:", error);
      }
    },

    async sendModoEco(deviceId) {

      this.sendCommand(deviceId, 'on', 'switch');

      this.sendCommand(deviceId, 'setAirConditionerMode', 'airConditionerMode',['cool']);
      
      this.sendCommand(deviceId, 'setAcOptionalMode', 'custom.airConditionerOptionalMode',['smart']);

      this.sendCommand(deviceId, 'setFanOscillationMode', 'fanOscillationMode',['vertical']);
      await this.sleep(5000); 
      console.log("aguardando vertical")

      this.sendCommand(deviceId, 'setFanOscillationMode', 'fanOscillationMode',['horizontal']);
      console.log("aguardando horizontal")
      await this.sleep(5000); 

      this.sendCommand(deviceId, 'setFanOscillationMode', 'fanOscillationMode',['fixed']);
      console.log("stop")

      this.sendCommand(deviceId, 'setFanMode', 'airConditionerFanMode', ['low'])

    },


  },
  mounted() {
    //this.fetchDevices();
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  text-align: center;
  color: #2c3e50;
  margin-top: 20px;
}
#resp {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  text-align: left;
  color: #2c3e50;
  margin-top: 20px;
}

button {
  margin: 5px;
  padding: 10px 20px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

button:hover {
  background-color: #0056b3;
}

</style>
