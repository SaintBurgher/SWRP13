<template>
  <div>

    <!-- Errors -->
    <div v-if="invalid" class="center">
      <span class="red" v-if="nocons">Error: No scanner bed detected!</span>
      <span class="red" v-else-if="!occupied">No occupant detected.</span>
      <span class="red" v-else-if="ipc">Error: Object in scanner bed interfering with sensor array.</span>
      <span class="red" v-else-if="noscan">Error: No diagnostics profile installed for this species.</span>
      <span class="red" v-else>Error: Unknown error.</span>
    </div>

    <!-- Health Display -->
    <table class="bodyscanner" v-else>
      <div class="row">

        <!-- Left Side -->
        <div class="column">
          <h3>Patient Status</h3>
          <hr>

          <!-- Patient Details -->
          <vui-group>
            <vui-group-item label="Name:">{{ name }}</vui-group-item>
            <vui-group-item label="Status:"><span :style="{color:consciousnessLabel(stat)}">{{ consciousnessText(stat) }}</span></vui-group-item>
            <vui-group-item label="Species:">{{ species }}</vui-group-item>
            <vui-group-item label="Brain Activity:"><vui-progress :class="progressClass(brain_activity)" :value="brain_activity">{{brain_activity}}%</vui-progress></vui-group-item>

            <!-- Trauma -->
            <template v-if="stat < 2">
              <vui-group-item label="Physical Trauma:"><span :style="{color:damageLabel(bruteLoss)}">{{ bruteLoss }}</span></vui-group-item>
              <vui-group-item label="Oxygen Deprivation:"><span :style="{color:damageLabel(oxyLoss)}">{{ oxyLoss }}</span></vui-group-item>
              <vui-group-item label="Organ Failure:"><span :style="{color:damageLabel(toxLoss)}">{{ toxLoss }}</span></vui-group-item>
              <vui-group-item label="Burn Severity:"><span :style="{color:damageLabel(fireLoss)}">{{ fireLoss }}</span></vui-group-item>
            </template>
          </vui-group>
          <hr>

          <!-- Status Effects -->
          <vui-group>
            <vui-group-item label="Radiation Level:">{{ Math.round(rads) }}</vui-group-item>
            <vui-group-item label="Genetic Damage:">{{ cloneLoss }}</vui-group-item>
            <vui-group-item label="Est. Paralysis Level:">{{ paralysis }}<span v-if="paralysis"> (~{{ Math.round(paralysis/4) }} Seconds Left)</span></vui-group-item>
            <vui-group-item label="Body Temperature:">{{bodytemp}} K (~ {{Math.round(bodytemp - 273.15)}} C)</vui-group-item>
          </vui-group>

          <!-- Blood -->
          <hr>
          <h3>Blood Status</h3>
          <hr>
          <vui-group>
            <vui-group-item label="BP:" :style="{color:getPressureClass(blood_pressure_level)}"> {{blood_pressure}} </vui-group-item>
            <vui-group-item label="Blood Oxygenation:"><vui-progress :class="progressClass(brain_activity)" :value="Math.round(blood_o2)">{{Math.round(blood_o2)}}%</vui-progress></vui-group-item>
            <vui-group-item label="Blood Volume:"><vui-progress :class="progressClass(brain_activity)" :value="Math.round(blood_volume)">{{Math.round(blood_volume)}}%</vui-progress></vui-group-item>
            <vui-group-item label="Inaprovaline:" v-if="Math.round(norepiAmt)"> {{ Math.round(norepiAmt) }} unit(s)</vui-group-item>
            <vui-group-item label="Soporific:" v-if="Math.round(soporAmt)"> {{ Math.round(soporAmt) }} unit(s)</vui-group-item>
            <vui-group-item label="Bicaridine:" v-if="Math.round(bicardAmt)"> {{ Math.round(bicardAmt) }} unit(s)</vui-group-item>
            <vui-group-item label="Dermaline:" v-if="Math.round(dermAmt)"> {{ Math.round(dermAmt) }} unit(s)</vui-group-item>
            <vui-group-item label="Dexalin:" v-if="Math.round(dexAmt)"> {{ Math.round(dexAmt) }} unit(s)</vui-group-item>
            <vui-group-item label="Thetamycin:" v-if="Math.round(thetaAmt)"> {{ Math.round(thetaAmt) }} unit(s)</vui-group-item>
            <vui-group-item label="Other:" v-if="Math.round(otherAmt)"> {{ Math.round(otherAmt) }} unit(s)</vui-group-item>
          </vui-group>
        </div>

        <!-- Right Side -->
        <div class="column">

          <!-- Missing Organs -->
          <template v-if="hasmissing">
            <div v-for="organ in missingparts" :key="organ.name">
              <vui-group-item> {{ organ }} </vui-group-item>
            </div>
          </template>

          <!-- Internal Organs -->
          <h3>Internal Organ Status</h3>
          <hr>
          <table class="injury">
            <template v-if="hasinternalinjury">
              <tr>
                <th>Organ</th>
                <th>Trauma</th>
                <th>Wounds</th>
                <th>Location</th>
              </tr>
            </template>
            <template v-else>
              <tr>
                <th><span style="color:LimeGreen">The occupant has no internal injuries.</span></th>
              </tr>
            </template>
            <tr v-for="organ in organs" :key="organ.name">
              <template v-if="organ.damage != 'None' || organ.hasWounds">
                <td> {{ organ.name }} </td>
                <td><span :style="{color:damageLabel(organ.damage)}"> {{ organ.damage }}</span></td>
                <td><div v-for="wound in organ.wounds" :key="wound.name" style="color:Tomato"> {{ wound }} </div></td>
                <td> {{ organ.location }} </td>
              </template>
            </tr>
          </table>

          <!-- External Organs -->
          <hr>
          <h3>External Bodypart Status</h3>
          <hr>
          <table class="injury">
            <template v-if="hasexternalinjury">
              <tr>
                <th>Organ</th>
                <th>Physical / Burn Trauma</th>
                <th>Wounds</th>
              </tr>
            </template>
            <template v-else>
              <tr>
                <th><span style="color:LimeGreen">The occupant has no external injuries.</span></th>
              </tr>
            </template>
            <tr v-for="organ in bodyparts" :key="organ.name">
              <template v-if="organ.bruteDmg != 'None' || organ.burnDmg != 'None' || organ.hasWounds">
                <td> {{ organ.name }} </td>
                <td><span :style="{color:damageLabel(organ.bruteDmg)}"> {{ organ.bruteDmg }} </span> / <span :style="{color:damageLabel(organ.burnDmg)}"> {{ organ.burnDmg }} </span></td>
                <td><div v-for="wound in organ.wounds" :key="wound.name" style="color:Tomato"> {{ wound }} </div></td>
              </template>
            </tr>
          </table>
        </div>

        <h3>Actions</h3>
        <vui-button class="center" :params="{print: 1}">Print Report</vui-button>
        <vui-button class="center" :params="{eject: 1}">Eject Occupant</vui-button>

      </div>
    </table>
  </div>
</template>


<script>
  export default {
    data() {
      return this.$root.$data.state;
    },
    methods: {
    consciousnessLabel(value) {
      switch (value) {
        case 0:
          return "LimeGreen"
        case 1:
          return "OrangeRed"
        case 2:
          return "Crimson"
        }
      },
      consciousnessText(value) {
      switch (value) {
        case 0:
          return "Conscious"
        case 1:
          return "Unconscious"
        case 2:
          return "DEAD"
        }
      },
    progressClass(value) {
      if (value <= 50) {
        return "bad"
      }
      else if (value <= 90) {
        return "average"
      }
      else {
        return "good"
      }
    },
    brainText(value) {
      switch (value) {
        case 0:
          return "None, patient is braindead"
        case -1:
          return "ERROR - Nonstandard biology"
        default:
          return value.toString().concat("%");
        }
      },
    damageLabel(value) {
      if(value == "extreme" || value < 20) {
        return "Crimson"
      }
      else if(value == "severe" || value < 40) {
        return "OrangeRed"
      }
      else if(value == "significant" || value < 60) {
        return "Tomato"
      }
      else if(value == "moderate" || value < 80) {
        return "Orange"
      }
      else if(value == "minor" || value < 100) {
        return "LawnGreen"
      }
      else {
        return "LimeGreen"
      }
    },
    getPressureClass(tpressure) {
      switch (tpressure) {
        case 1:
          return "Crimson"
        case 2:
          return "LimeGreen"
        case 3:
          return "LawnGreen"
        case 4:
          return "Crimson"
        default:
          return "LightSkyBlue"
      }
    },
  }
}
</script>

<style lang="scss" scoped>
.bodyscanner {
  display: table;
  width: 100%;

  .header {
    display: table-row;
    .header-item, .item {
      display: table-cell;
      padding: 1px;
      vertical-align: middle;
    }
  }

  .column {
    float: left;
    width: 45%;
    margin: 1%;
    padding: 10px;
    outline-style: ridge;
    outline-color: black;
    background-color: rgba(0, 0, 0, 0.4);
    height: 500px;
    overflow: auto;
  }

  /* Clear floats after the columns */
  .row:after {
    content:"";
    display: table;
    clear: both;
  }
}

.left {
  text-align: left;
}

.center {
  text-align: center;
}

.right {
  text-align: right;
}

table .injury {
  border-spacing: 20px 0;
}

td, th {
  text-align: center;
}
</style>
