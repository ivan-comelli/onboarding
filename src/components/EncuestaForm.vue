<template>
    <div class="survey">
      <h1>Encuesta: Perfiles Conductuales</h1>
      <div class="items" v-for="(item, index) in survey" :key="index">
        <h5>{{ index }}</h5>
        <div>
          <h3>{{ item.thread.title }}</h3>
          <p>{{ item.thread.instructions  }}</p>
        </div>
        <div class="scenarios" v-if="answersOptions[index] == null">
          <div class="instance" v-for="(instance, idx) in item.thread.scenarios" :key="idx">
            <h4 class="title">{{ instance.title  }}</h4>
            <label>
              <input
                type="radio"
                :name="'pregunta' + index"
                :value="instance"
                v-model="answersOptions[index]"
                @change="pagesIndex[index] = 0"
              />
              {{ instance.narrative }}
            </label>
          </div>
        </div>
        <div class="questions" v-else-if="pagesIndex[index] != null">  
          <div>
            <h4>
              {{ answersOptions[index].questions[pagesIndex[index]].content }}
            </h4>
            
            <div class="answer">
              {{ answersOptions[index].questions[pagesIndex[index]].answers[answersOptions[index].questions[pagesIndex[index]].index].content }}
              <vue-slider v-model="answersOptions[index].questions[pagesIndex[index]].index" :min="0" :max="3" />
              <div class="indicators">
                <span @click="answersOptions[index].questions[pagesIndex[index]].index = 0">Bueno</span>
                <span @click="answersOptions[index].questions[pagesIndex[index]].index = 1">Casi Bueno</span>
                <span @click="answersOptions[index].questions[pagesIndex[index]].index = 2">Regular</span>
                <span @click="answersOptions[index].questions[pagesIndex[index]].index = 3">Malo</span>
              </div>
            </div>
            
          </div>
          <div class="actions">
            <button @click="pagesIndex[index] > 0 ? pagesIndex[index] -= 1 : answersOptions[index] = null">Atras</button>
            <button @click="pagesIndex[index] < 2 ? pagesIndex[index] += 1 : null">Siguiente</button>
          </div>
        </div>
      </div>
      <button @click="submitSurvey">Enviar Encuesta</button>
      <apexchart width="500" type="bar" :options="options" :series="series" />
    </div>
  </template>
  
  <script>
  import surveyJson from '../data.json';
  import valuesJson from '../model.json';
  import VueApexCharts from "vue3-apexcharts";
  import VueSlider from 'vue-slider-component'
  import 'vue-slider-component/theme/antd.css'

  export default {
    components: {
      apexchart: VueApexCharts,
      VueSlider
    },
    data() {
      return {
        pagesIndex: [],
        options: {
          bar: {
            horizontal: true
          }
        },
        series: [{
          data: [{
            x: 'Self Esteem',
            y: 100
          }, {
            x: 'Caution',
            y: 100
          }, {
            x: 'Cognition',
            y: 100
          }, {
            x: 'Discipline',
            y: 100
          }, {
            x: 'Conformism',
            y: 100
          }, {
            x: 'Flexibility',
            y: 100
          }]
        }],
        survey: [], // Aquí se almacenará el JSON de la encuesta
        mapValues: [],
        answersOptions: [], // Aquí se almacenarán las respuestas seleccionadas
      };
    },
    mounted() {
      this.survey = surveyJson;
      this.mapValues = valuesJson;
      this.answers = new Array(this.survey.length).fill('');

      for (let prop in this.mapValues) {
        let obj = this.mapValues[prop];
        let serieW = [];
        let sumWeight = 0;

        // Recorremos el objeto una sola vez para obtener los pesos y la suma de los pesos.
        for (let propBias in obj) {
          let objBias = obj[propBias];
          if (objBias.weight) {
            serieW.push(objBias.weight);
            sumWeight += Math.abs(objBias.weight);
          }
        }

        // Calculamos el valor inicial usando la suma de los pesos.
        obj.initValue = Number(this.calculateBalancedInit(serieW).toFixed(2));

        // Recorremos el objeto nuevamente para calcular las proporciones.
        for (let propBias in obj) {
          let objBias = obj[propBias];
          if (objBias.weight) {
            objBias.proportion = Number((objBias.weight / sumWeight).toFixed(2));
          }
        }
      }
    },
    methods: {
      calculateBalancedInit(ponderaciones) {  
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
        for (let prop in this.mapValues) {
          let obj = this.mapValues[prop];
          let sumFinalValue = obj.initValue;
          for (let propSesgo in obj) {
            let objBias = obj[propSesgo];
            if (objBias.weight) {
              sumFinalValue += objBias.proportion * objBias.intensity
            }
          }
          console.log(Number(sumFinalValue.toFixed(2))*100)
          obj.finalValue = Number(sumFinalValue.toFixed(2))*100;
          this.series[0].data[index].y = Number(sumFinalValue.toFixed(2))*100;
          index += 1;
        }
      },
      submitSurvey() {
        this.answersOptions.map((item) => {
          for (const key in item.biases) {
            const bias = item.biases[key];
            switch(key) {
              case "overconfidence":
                  this.mapValues.selfEsteem.overconfidence.intensity = bias.factor;
                  break;
              case "selfAttribution":
                  // Asumimos que "selfAttribution" es un sesgo bajo "selfEsteem"
                  this.mapValues.selfEsteem.selfAttribution.intensity = bias.factor;
                  break;
              case "statusQuo":
                  this.mapValues.caution.statusQuo.intensity = bias.factor;
                  break;
              case "regretAversion":
                  // Asumimos que "regretAversion" es un sesgo bajo "caution"
                  this.mapValues.caution.regretAversion.intensity = bias.factor;
                  break;
              case "representativeness":
                  this.mapValues.cognition.representativeness.intensity = bias.factor;
                  break;
              case "anchoringEffect":
                  this.mapValues.cognition.anchoringEffect.intensity = bias.factor;
                  break;
              case "mentalAccounting":
                  this.mapValues.cognition.mentalAccounting.intensity = bias.factor;
                  break;
              case "confirmationBias":
                  this.mapValues.cognition.confirmationBias.intensity = bias.factor;
                  break;
              case "selfControl":
                  this.mapValues.discipline.selfControl.intensity = bias.factor;
                  break;
              case "procrastination":
                  this.mapValues.discipline.procrastination.intensity = bias.factor;
                  break;
              case "endowmentEffect":
                  this.mapValues.conformism.endowmentEffect.intensity = bias.factor;
                  break;
              case "herdBehavior":
                  this.mapValues.conformism.herdBehavior.intensity = bias.factor;
                  break;
              case "dispositionEffect":
                  this.mapValues.flexibility.dispositionEffect.intensity = bias.factor;
                  break;
              case "lossAversion":
                  this.mapValues.flexibility.lossAversion.intensity = bias.factor;
                  break;
              case "instantGratification":
                  // Asumimos que "instantGratification" es un sesgo bajo "flexibility"
                  this.mapValues.flexibility.instantGratification.intensity = bias.factor;
                  break;
              default:
                  console.warn(`Bias '${key}' not defined in the model.`);
                  break;
            }
        }

        });
        this.calculateFinalValues();
      }
    }
  };
</script>
  
<style scoped>
  .survey {
    max-width: 600px;
    margin: 0 auto;
    padding: 20px;
    border: 1px solid #ccc;
    border-radius: 5px;
  }
  .encusurveyesta h1 {
    text-align: center;
  }
  .survey h3 {
    margin-top: 20px;
    text-align: left;
  }
  .survey h4 {
    text-align: left;
    height: 2em;
  }
  .survey label {
    display: block;
    margin-bottom: 10px;
    text-align: left;
    padding: 0 1em;
  }
  .survey button {
    display: block;
    padding: 10px 20px;
    color: #007BFF;
    background-color: none;
    border: none;
    border-radius: 5px;
    cursor: pointer;
  }
  .questions {
    background: #d1d1d1;
    padding: 2em;
  }
  p {
    text-align: left;
  }
  .scenarios {
    background: #d9d9d9;
    padding: 1em;
  }
  .instance {
    margin-bottom: 3em;
  }
  .items {
    display: flex;
    flex-direction: column;
    gap: 1em;
    border-bottom: solid .1em #c3c3c3;
    padding-bottom: 2em;
  }
  .survey h5 {
    border-bottom: 1px solid #ccc;
    padding-bottom: 2em;
  }
  .actions {
    display: flex;
    justify-content: center;
    flex-direction: row;
    padding-top: 3em;

    gap: 5em;
  }
  .answer {
    display: flex;
    flex-direction: column;
    gap: 1em;
    color: #007071;
  }
  .indicators {
    display: flex;
    flex-direction: row;
    justify-content: space-between;
  }
  .indicators span { 
    color: white !important;
    padding: 1em 2em;
    border-radius: 1em;
  }
  .indicators span:nth-child(1) {
    background-color: rgb(0, 62, 196) !important;
  }
  .indicators span:nth-child(2) {
    background-color: rgb(49, 173, 0)!important;
  }
  .indicators span:nth-child(3) {
    background-color: rgb(255, 172, 18) !important;
  }
  .indicators span:nth-child(4) {
    background-color: rgb(194, 8, 8) !important;
  }
</style>
  