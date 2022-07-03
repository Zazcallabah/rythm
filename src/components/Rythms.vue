
<script setup lang="ts">
import { ref } from "vue"
import * as Tone from 'tone'

type Sequence = {
  ix: number;
  note: string;
  beats: boolean[];
  synth: string;
};
const synth = new Tone.Synth().toDestination();

const bitcount = (x:bigint) => {
  if (x === 0n) {
    return 1n;
  }
  let extra = 0n;
  if (x > 0xffffffffn) {
    x = x >> 32n;
    extra = 32n;
  }
  const count = BigInt(Math.floor(Math.log2(Number(x))) + 1);
  return count + extra;
}

const makeFlags = (set:bigint[]) : boolean[] => {
  const result = [];

  for (const n of set) {
    const b = bitcount(n);
    for (let mask = 1n << (b - 1n); mask >= 1n; ) {
      result.push((n & mask) > 0n);
      mask = mask >> 1n;
    }
  }
  return result;
}

const euclidianRythm = async (beats: number, length: number)=> {
  const empties = length - beats;

  const workset = [] as bigint[];
  for (let x = 0; x < beats; x++) {
    workset.push(BigInt(1));
  }
  for (let x = 0; x < empties; x++) {
    workset.push(BigInt(0));
  }

  while (workset.length > 1) {
    const last = workset[workset.length - 1];

    let tailLength = 0;
    for (let x = workset.length - 1; x >= 0 && workset[x] === last; x--) {
      tailLength++;
    }
    if (tailLength === 1 || tailLength === workset.length) {
      break;
    }
    workset.splice(workset.length - tailLength, tailLength);

    const b = bitcount(last);
    for (let x = 0; x < tailLength; x++) {
      const ix = x % workset.length;
      workset[ix] = (workset[ix] << b) | last;
    }
  }
  const result = makeFlags(workset);
  return result;
}

const playHead = ref(-1)
const notes = ref(["G4", "A4", "A#4", "C4", "D4", "E4", "F4", "G5", "A5", "A#5", "C5", "D5", "E5", "F5", "G6", "A6", "A#6", "C6", "D6", "E6", "F6"])


const play = () => {
  state.value = "playing"
  playHead.value = -1;
  Tone.Transport.bpm.value = 120;
  const synths = data.value.map((seq)=>{
    if( seq.synth = "fm" ){
      return new Tone.FMSynth().toDestination()
    }
    else return new Tone.AMSynth().toDestination()
  })
  Tone.Transport.scheduleRepeat((time) => {
    playHead.value = (playHead.value + 1) % length.value
    for( const ix in data.value ){
      const seq = data.value[ix]
      if( seq.note && seq.beats[playHead.value] ){
        synths[ix].triggerAttackRelease(seq.note, length.value+"n", time);
      }
    }
  }, length.value+"n");
  Tone.Transport.start();
}
const stop = () => {
  state.value = "stopped"
  Tone.Transport.stop();
  playHead.value = -1
}
const state = ref("")
const length = ref( 16 )
const error = ref("");
const data = ref( [] as Sequence[])
const synths = ref([])
const load = async ()=>{
      await Tone.start()
			data.value = []
			if (length.value > 0) {
				for (var i = 1; i <= length.value; i++) {
					let s2 = "";
					data.value.push( {
            ix: i,
            synth: "fm",
            beats: await euclidianRythm(i, length.value),
            note: ""
          })
				}
			} else error.value = "bad data"
      state.value = "stopped";
}
</script>

<template>
  <p>
    <button v-show="state ==='stopped'" v-on:click="play">▶️</button>
    <button v-show="state==='playing'" v-on:click="stop">⏹️</button>
  </p>
	<input v-model="length" /> <button v-on:click="load">Go</button>
  <p>{{error}}</p>

	<table>
    <tr v-for="(row,ri) in data" :key="row.ix">
      <td v-for="(cell,ci) in row.beats" :key="`cell-${ri}-${ci}`" :class="{ head: ci === playHead, beat: cell, free: !cell }">&nbsp;</td>
<td>
<select v-model="row.note">
  <option value="">n/a</option>
  <option v-for="note in notes" :key="note" :value="note">{{ note }}</option>
</select>

</td>
    </tr>
	</table>
</template>

<style scoped>
body {
  margin: 2rem;
}
#anim {
  float: right;
  width: 40vw;
}

.head {
  background: lightgreen !important;
}
.beat {
  border: 1px solid black;
  background: pink;
}

.free {
  border: 1px solid silver;
  background: white;
}
</style>
