<template>
    <div class="encuesta">
      <h1>Encuesta: Perfiles Conductuales</h1>
      <div class="items" v-for="(item, index) in encuesta" :key="index">
        <h3>{{ item.pregunta }}</h3>
        <div v-if="!item.flagResolve">
          <label v-for="(respuesta, idx) in item.respuestas" :key="idx">
            <input
              type="radio"
              :name="'pregunta' + index"
              :value="respuesta"
              v-model="respuestas[index]"
              @change="item.flagResolve = true"
            />
            {{ respuesta.respuesta }}
          </label>
        </div>
        <div class="afirmations" v-if="item.flagResolve">
            <b>{{ respuestas[index].respuesta }}</b>
            <div class="afirmations-options">
              <label v-for="(sesgo, idx) in respuestas[index]['sesgos']" :key="idx">
                <input 
                  type="checkbox"
                  :value="!sesgo.status"
                  v-model="sesgo.status"
                />
                {{ sesgo.afirmation }}
              </label>
            </div>
            <button @click="respuestas[index] = ''; item.flagResolve = false">Atras</button>
        </div>
      </div>
      <button @click="submitEncuesta">Enviar Encuesta</button>
      <apexchart width="500" type="bar" :options="options" :series="series"></apexchart>
    </div>
  </template>
  
  <script>
  import encuesta from '../data.json';
  import values from '../values.json';
  import VueApexCharts from "vue3-apexcharts";

  export default {
    components: {
      apexchart: VueApexCharts,
    },
    data() {
      return {
        options: {
          bar: {
            horizontal: true
          }
        },
        series: [{
          data: [{
            x: 'Attitude',
            y: 100
          }, {
            x: 'Targeting',
            y: 100
          }, {
            x: 'Impulsiveness',
            y: 100
          }, {
            x: 'Prudence',
            y: 100
          }, {
            x: 'Conformism',
            y: 100
          }, {
            x: 'Balance',
            y: 100
          }]
        }],
        encuesta: [], // Aquí se almacenará el JSON de la encuesta
        values: [],
        respuestas: [], // Aquí se almacenarán las respuestas seleccionadas
      };
    },
    mounted() {
      this.encuesta = encuesta;
      this.values = values;
      this.respuestas = new Array(this.encuesta.length).fill('');

      for (let prop in this.values) {
        let obj = this.values[prop];
        let serieW = [];
        let sumaPesos = 0;

        // Recorremos el objeto una sola vez para obtener los pesos y la suma de los pesos.
        for (let propSesgo in obj) {
          let objSesgo = obj[propSesgo];
          if (objSesgo.weight) {
            serieW.push(objSesgo.weight);
            sumaPesos += Math.abs(objSesgo.weight);
          }
        }

        // Calculamos el valor inicial usando la suma de los pesos.
        obj.initValue = Number(this.calcularX0(serieW).toFixed(2));

        // Recorremos el objeto nuevamente para calcular las proporciones.
        for (let propSesgo in obj) {
          let objSesgo = obj[propSesgo];
          if (objSesgo.weight) {
            objSesgo.proportion = Number((objSesgo.weight / sumaPesos).toFixed(2));
          }
        }

        console.log(obj);
      }
    },
    methods: {
      calcularX0(ponderaciones) {  
        let conjuntoNegativo = 0;
        let sumaPonderaciones = 0;
        for (let i = 0; i < ponderaciones.length; i++) {
            let ponderacionAbsoluta = Math.abs(ponderaciones[i]);
            sumaPonderaciones += ponderacionAbsoluta;
            if(ponderaciones[i] < 0) {
              conjuntoNegativo += ponderaciones[i];
            }
        }
        let X0 = Math.abs(conjuntoNegativo / sumaPonderaciones);        
        return X0;
      },
      calculateFinalValues() {
        let index = 0;
        for (let prop in this.values) {
          let obj = this.values[prop];
          let sumFinalValue = obj.initValue;
          for (let propSesgo in obj) {
            let objSesgo = obj[propSesgo];
            if (objSesgo.weight) {
              sumFinalValue += objSesgo.proportion * objSesgo.intensity
            }
          }
          console.log(Number(sumFinalValue.toFixed(2))*100)
          obj.finalValue = Number(sumFinalValue.toFixed(2))*100;
          this.series[0].data[index].y = Number(sumFinalValue.toFixed(2))*100;
          index += 1;
        }
      },
      submitEncuesta() {
        this.respuestas.map((respuesta) => {
          const sesgos = respuesta.sesgos;
          sesgos.map((sesgo) => {
            switch(sesgo.tipo) {
              case "overconfidence":
                this.values.attitude.overconfidence.intensity = sesgo.factor;
              break;
              case "statusQuo":
                this.values.attitude.statusQuo.intensity = sesgo.factor;
              break;
              case "overreaction":
                this.values.targeting.overreaction.intensity = sesgo.factor;
              break;
              case "framing":
                this.values.targeting.framing.intensity = sesgo.factor;
              break;
              case "representativeness":
                this.values.targeting.representativeness.intensity = sesgo.factor;
              break;
              case "availabilityBias":
                this.values.targeting.availabilityBias.intensity = sesgo.factor;
              break;
              case "selfControl":
                this.values.impulsiveness.selfControl.intensity = sesgo.factor;
              break;
              case "endowmentEffect":
                this.values.prudence.endowmentEffect.intensity = sesgo.factor;
              break;
              case "lossAversion":
                this.values.prudence.lossAversion.intensity = sesgo.factor;
              break;
              case "mentalAccounting":
                this.values.prudence.mentalAccounting.intensity = sesgo.factor;
              break;
              case "herdBehavior":
                this.values.conformism.herdBehavior.intensity = sesgo.factor;
              break;
              case "gamblersFallacy":
                this.values.balance.gamblersFallacy.intensity = sesgo.factor;
              break;
              case "dispositionEffect":
                this.values.balance.dispositionEffect.intensity = sesgo.factor;
              break;
              case "anchoring":
                this.values.balance.anchoring.intensity = sesgo.factor;
              break;
            }
          });
        });
        this.calculateFinalValues();
      }
    }
  };
  </script>
  
  <style scoped>
  .encuesta {
    max-width: 600px;
    margin: 0 auto;
    padding: 20px;
    border: 1px solid #ccc;
    border-radius: 5px;
  }
  .encuesta h1 {
    text-align: center;
  }
  .encuesta h3 {
    margin-top: 20px;
  }
  .encuesta label {
    display: block;
    margin-bottom: 10px;
    text-align: left;
    margin-left: 2em;
  }
  .encuesta button {
    display: block;
    margin: 20px auto;
    padding: 10px 20px;
    background-color: #007BFF;
    color: #fff;
    border: none;
    border-radius: 5px;
    cursor: pointer;
  }
  .afirmations {
    background: #f7f7f7;
    padding: 2em;
  }
  .afirmations-options {
    padding: 2em 0em;
  }
  .items {
    display: flex;
    flex-direction: column;
    gap: 1em;
    border-bottom: solid .1em #c3c3c3;
    padding-bottom: 2em;
  }
  b {
    color: #30316d;
  }
  </style>
  