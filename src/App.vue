
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
            <li v-for="device in filteredDevices" :key="device.deviceId">
              <h2>{{ device.label || device.name }}</h2>
              <p>ID: {{ device.deviceId }}</p>
              
              <!--p>Capacidades: {{ device.capabilities.join(", ") }}</p-->

              <!-- Botões para comandos -->
              <button @click="sendCommand(device.deviceId, 'on', 'switch')">Ligar</button>
              <button @click="sendCommand(device.deviceId, 'off', 'switch')">Desligar</button>
              <!--button @click="sendCommand(device.deviceId, 'setCoolingSetpoint', 'thermostatCoolingSetpoint', [22])">
              Ajustar para 22°C
              </button-->

              <button @click="sendModoEco(device.deviceId)">
              Modo Eco
              </button>

              <button @click="getDeviceConfig(device.deviceId, '')">Consulta CFGs</button>

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
      baseUrl : "https://samsung-integracao-995c5632d56a.herokuapp.com", // API que comunicara com o SmartThings 
    };
  },

  computed: {
    filteredDevices() { 
      return this.devices.filter(item => item.label === "Ar suite");
    },
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
        this.loading = true;
        this.logon=false;

        response = await axios.get(`${this.baseUrl}/devices`,{ headers: {pwd:this.pwd.trim()}, });
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
        this.loading = true;
        const response = await axios.post(`${this.baseUrl}/device/${deviceId}/command`, {          
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
      this.loading = false;
    },

    // CARREGA INDORMACOES DO DISPOSITIVO
    async getDeviceConfig(deviceId, command) {
      //console.log(deviceId);
      //console.log(command);
      this.loading = true;

      if (command.trim() == '')
        command = "all";

      const url = `${this.baseUrl}/device/${deviceId}/get/${command}`;
      //const url = `http://localhost:3001/device/${deviceId}/get/${command}`;
      console.log(url)

      try {
        const response = await axios.get(url,{headers: {pwd:this.pwd.trim()},});

        this.deviceConfig = response.data.data || {};
        console.log("Configurações do Dispositivo:", this.deviceConfig);
      } catch (error) {
        console.error("Erro ao recuperar configurações do dispositivo:", error);
      }
      this.loading = false;
    },

    async sendModoEco(deviceId) {
      this.loading = true;
      this.sendCommand(deviceId, 'on', 'switch');

      this.sendCommand(deviceId, 'setLighting', 'samsungce.airConditionerLighting',['off']);

      await this.sleep(1000);
      this.result = "ajustando modo Arrefecer";
      this.sendCommand(deviceId, 'setAirConditionerMode', 'airConditionerMode',['cool']);
      
      await this.sleep(1000);
      this.result = "ajustando modo Eco";
      this.sendCommand(deviceId, 'setAcOptionalMode', 'custom.airConditionerOptionalMode',['smart']);     
      
      // aguardo ar se ajustar ao modo definido
      await this.sleep(15000);
      this.result = "Set temperatura";
      this.sendCommand(deviceId, 'setCoolingSetpoint', 'thermostatCoolingSetpoint', [24]);
      await this.sleep(1000);


      // AJUSTE DE FANS
      // sleeps ate chegar a posicao de minha preferencia
      // *****************************************************
      this.result = "aguardando pas de ocilação";
      this.sendCommand(deviceId, 'setFanOscillationMode', 'fanOscillationMode',['vertical']);
      await this.sleep(4600); 
      
      this.result = "aguardando pas horizontais";
      this.sendCommand(deviceId, 'setFanOscillationMode', 'fanOscillationMode',['horizontal']);
      await this.sleep(7500); 

      this.result = "parando Ocilacao";
      this.sendCommand(deviceId, 'setFanOscillationMode', 'fanOscillationMode',['fixed']);
      
      await this.sleep(1000);
      this.result = "setando Fans no minimo";
      this.sendCommand(deviceId, 'setFanMode', 'airConditionerFanMode', ['low']);
      // *****************************************************

      await this.sleep(2000);
      this.result = '';
      this.loading = false;

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
