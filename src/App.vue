<script setup lang="js">

import { ref, onMounted } from "vue";
import * as Tone from "tone";

import { api as fullscreen } from 'vue-fullscreen'
import { LocalStorage  } from 'web-browser-storage'
import exportFromJSON from 'export-from-json'
import { useFileDialog } from '@vueuse/core'
import { createConfirmDialog, DialogsWrapper  } from 'vuejs-confirm-dialog'
import { compressToBase64, decompressFromBase64 } from 'lz-string'


import ModalDialog from './ModalDialog.vue'
import Note from "./Note.vue";
import Piano from "./Piano.vue";


// Si la paramètre data est passé dans l'url, on récupère les donnés du paramètre pour les charger dans le morceau
// NE MARCHE PAS. URL TROP LONGUE
let urlParam = new URLSearchParams(window.location.search);
// console.log(urlParam.has('data')); // true
// console.log(urlParam.get('data')); // "MyParam"



// On importe un fichier contenant un json compressé en base64
// Puis on rentre ces donnés dans le morceau
const { files, open, reset, onChange } = useFileDialog({
  accept: '*.json/*', // Set to accept only image files
  directory: false, // Select directories instead of files if set true
})

onChange((files) => {
  files[0].text().then((response) => {
    let exportData = compressToBase64(response)
    // console.log(exportData)
    let data = JSON.parse(decompressFromBase64(response))
    resetPaterns()
    voicesPatterns.value = data.voices
    drumsPatterns.value = data.drums
    song.value = data.song
    bpm.value = data.bpm
    melodyPatterns.value = data.melody
    initiatedByStorage = true
    updateSong()
  })
})


// L'octace en cours du piano.
// L'indicateur indique la précédente et la suivante, ce qui permet de savoir quelle est celle actuelle
var octave = ref(4);

// Stockage du navigateur initialisé pour stocker à chaque stop les données du morceau
const storage = new LocalStorage()

// Booléen pour savoir si le contexte audio est démarré. Celà nécessite une action de l'utilisateur (clic sur un élément)
var audioContextActivated = false;

// Booléen pour garder l'état du playback
var isPlaying = false;

// Booléen pour savoir si le Transport de ToneJS a été crée, à la première lecture une fois l'appli chargée, ou rechargée
var transportCreated = false;


// Démarre le contexte Audio. Déclenché par un clic
async function startAudioContext() {
  await Tone.start();
  audioContextActivated = true;
}


// Pour afficher le statut du playback
let playStatus = ref("Play")

// Le nombre de beats passés depuis le début de la lecture. Revient à 0 à la fin de la boucle de patterns.
// Initialisé à zéro au début du morceau, à partir du pattern selectionner comme début
// Ex: Si on commence au tout début, le premier beat de la seconde mesure sera 3, si on commence à la seconde mesure ça sera 0
var beat = ref(0);

// Le tempo
let bpm = ref(130)


// Les différentes longueurs de note disponibles
// possibleRynthms correspond à la Font Rythm qui affiche des notes à la place des lettres
// La translation est l'équivalent de ces valeurs de durée pour ToneJS
let rythmes = ref([])
let possibleRythme = ["z", "s", "e", "q", "h", "w", "b"]
let possibleRythmeTranslation = ["32n", "16n", "8n", "4n", "2n", "1m", "2m"]


// On attribue les durées possible à une variable
// On récupèrera la durée pout ToneJS car la liste est organisée de la même façon
for (let i = 0; i < possibleRythme.length; i++) {
  rythmes.value.push(possibleRythme[i])
}

// La durée de note sélectionnée actuellement.
// C'est l'indice de la liste de rythmes
let choosedRythm = ref(0)


// On incrément ou décrémente l'indice des durées disponible ce qui récupère la prochaine durée dans la liste rythmes.
function changeRythm(mode) {
  if (mode === 'up') {
    choosedRythm.value = Math.min(choosedRythm.value +1, possibleRythme.length -1)
  } else if (mode === 'down') {
    choosedRythm.value = Math.max(choosedRythm.value -1, 0)
  }

}


// Le contenu du presse papier interne du logiciel
// Type est le mode de copie: Drum, Voices ( ou Melody 1 ou 2 => À IMPLÉMENTER )
let clipboard = ref({'type': null, 'data': null})


// Booléen pour savoir si le mode qui suite la tête de lecture est actif
// Fonction pour le toggle avec un bouton
let followHead = ref(true)
function toggleFollow() {
  followHead.value ^= true
  updateSong()
}


// Fonction pour savoir si le mode qui permet d'entendre les notes jouées sur le piano quand la lecture est en cours,
// et empèche les ? d'être remplis par les notes jouées sur le piano
let soundDuringPlay = ref(false)
function toggleSoundDuringPlay() {
  soundDuringPlay.value ^= true
}


// Le nombre de beats par pattern. 
let patternsNumber = 16


// Régler le tempo
function setBpm(mode) {
  if (mode === 'up') {
    bpm.value = bpm.value +5
  } else {
    bpm.value = bpm.value -5
  }
  Tone.getTransport().bpm.value = bpm.value

}


// Booléen pour savoir si le mode qui boucle le pattern en cours est actif
// Celà va régler le song à uniquement ce pattern/

// BUG
// BUG
// BUG LE BEAT%16 N'EST PAS CORRECT
// BUG
// BUG
let singlePatternPlay = ref(false)
function toggleSinglePatternPlay() {
  singlePatternPlay.value ^= true
  beat.value = beat.value%16
  updateSong()
}


// Contient le liste des patterns à jouer, dans
// l'ordredans lequels ils sont joués
// Initialisé avec uniquement le premier pattern
let song = ref([])
song.value = [0]

// Le pattern affiché actuellement
let visiblePattern = ref(0);

// Contient les donnés musicales (notes, durées)
// Un élement par instuments.
// Un dictionnaire pour chaque beat de chaque intrument
// Contenant la note et la durée pour les Voices
// Ou uniquement la variation de son pour les drums
let voicesPatterns = ref([]);
let drumsPatterns = ref([])

// Contient les patterns de Voices et Drum contenus dans la song.
// Différents des variables précédents car les patterns peuvent se répéter
// Celle là contiennent l'ensemble de ce qui sera jouer
let completeSong = [];
let completeDrumsSong = [];

// Le patern en train d'e^tre joué
let currentSongPattern = ref(0)


// On initialise les variables précédents.
// Les notes "-" sont des notes vides, elles sont ignorées à la lecture et affichées vide.
// On ne peut pas utiliser une valeur vide car il faut afficher un charactère dans la note sinon 
// L'affichage bugue

// EN FAIT SI
// IL SUFFIT DE FAIRE UN IF
// PAS TRÈS GRAVE MAIS AUGMENTE LA TAILLE DU JSON

// Bool pour savoir si on est en mode song
// Dans ce cas le reset efface l'ordre dans lesquel les pattterns sont joué
// Mais pas les données du morceaux
let songSetupEnCours = ref(false)


// Demande confirmation avec de supprimer
function resetPaterns() {
  if (songSetupEnCours.value===false) {
  Tone.getTransport().stop();
  isPlaying = false;
  playStatus.value = "Play"
  beat.value = 0;
  completeSong = [];
  completeDrumsSong = [];
  visiblePattern.value = 0;
  voicesPatterns.value = [];

  // Pour chaque beat dans le pattern
  // On stocke pour chaque instumnent
  
  //
  // 4 HARCODÉ (Nombre d'instruments)
  //
  // ET 16 HARDCODÉ (Nombre de patterns dispo)
  //

  // et pour chaque pattern disponible la note par défaut
  // qui vaut pour vide avec une durée arbitraire

  //
  // C'EST UN PEU À L'ENVERS COMME BOUCLE
  //

  for (let i = 0; i < patternsNumber; i++) {
    voicesPatterns.value.push([]);
    for (let j = 0; j < 4; j++) {
      voicesPatterns.value[i].push([]);
      for (let k = 0; k < 16; k++) {
        voicesPatterns.value[i][j].push({note: "-", duree: '16n'})
      }
    }
  }


  // Pareil pour les drums. Ici pas besoin d'indication de durée
  drumsPatterns.value = []

  for (let i = 0; i < patternsNumber; i++) {
    drumsPatterns.value.push([]);
    for (let j = 0; j < 4; j++) {
      drumsPatterns.value[i].push([])
      for (let k = 0; k < 16; k++) {
        drumsPatterns.value[i][j].push("-")
      }
    }
  }  

  // Pour les mélodies c'est pas pareil
  // Comme il y a 2 instruments mélodiques on est
  // Obligés de stocker 2 listes complètes, car
  // ce ne sont pas des patterns mais une liste linéaire
  // On regroupe les beats dans des blcos de 4 mesures
  // Comme 4 mesures sont affichés en même temps
  // Celà permet de garder en mémoire l'indice des
  // Mesures en cours d'affichage

  //
  // 512 HARCODÉ (Durée maximale de la mélodie en beat)
  // 16 AUSSI (Nombre de beat par pattern)
  // ET 4 (Nombre de mesure dans un bloc)
  //


  melodyPatterns.value = []

    for (let i = 0; i < 2; i++) {
      melodyPatterns.value.push([])

      for (let j = 0; j < 512; j++) {
        melodyPatterns.value[i].push([])
      for (let k = 0; k < 4; k++) {
        melodyPatterns.value[i][j].push([])
        for (let l = 0; l < 16; l++) {
        melodyPatterns.value[i][j][k].push({note: "-", duree: '16n'})
      }
      }
    }
    }

}

  // On stocke les données réinitialisés dans le navigateur

  //
  // PEUT ÊTRE PAS UTILE? EN CAS D'ERREUR UN REFRESH POURRAIT ANNULER LE RESET
  // LES DONNÉS SONT STOCKÉS À CHAQUE STOP
  //

  song.value = [0]
  storage.set('voices', voicesPatterns.value)
  storage.set('drums', drumsPatterns.value)
  storage.set('song', [0])
  storage.set('bpm', 130)
  storage.set('melody', melodyPatterns.value)

}



// Dialogue de confirmation pour reset le morceau au l'rodre d'nchainement des patterns

const dialog_reset_patterns = createConfirmDialog(ModalDialog)

dialog_reset_patterns.onConfirm(() => {
  resetPaterns()
  dialog_reset_patterns.close()
})

dialog_reset_patterns.onCancel(() => {
  dialog_reset_patterns.close()
})


function askResetPaterns() {
  dialog_reset_patterns.reveal()
}


// Reverb appliquée à tous les instruments
let masterReverb = new Tone.Reverb({
  decay : 1.5 ,
  preDelay : 0.01,
  wet: 0.2
}).toDestination()




function setInstruments() {
let instruments = [
  new Tone.DuoSynth(),
  new Tone.PolySynth(
    {voices: 4,
     voice: Tone.FMSynth}),
  new Tone.MonoSynth(),
  new Tone.MonoSynth()
]

let synthJSON = {
   vibratoAmount: 0.4,
   vibratoRate: 5,
   harmonicity: 1.5,
   volume: -6,
   voice0: {
      volume: -12,
      portamento: 0,
      oscillator: {
         type: 'sine'
      },
      filterEnvelope: {
         attack: 0.01,
         decay: 0,
         sustain: 1,
         release: 0.5
      },
      envelope: {
         attack: 0.01,
         decay: 0,
         sustain: 1,
         release: 0.5
      }
   },
   voice1: {
      volume: -10,
      portamento: 0,
      oscillator: {
         type: 'triangle'
      },
      filterEnvelope: {
         attack: 0.01,
         decay: 0,
         sustain: 1,
         release: 0.5
      },
      envelope: {
         attack: 0.01,
         decay: 0,
         sustain: 1,
         release: 0.5
      }
   }
}

instruments[0].set(synthJSON);
instruments[0].connect(masterReverb)


synthJSON =  {
   harmonicity: 3,
   modulationIndex: 10,
   detune: 0,
   volume: -10,
   oscillator:{
      type: 'sine'
   },
   envelope:{
      attack: 0.002,
      decay: 0.01,
      sustain: 1,
      release: 0.5
   },
   modulation:{
      type: 'square'
   },
   modulationEnvelope:{
      attack: 0.004,
      decay: 0.005,
      sustain: 0.8,
      release: 0.8
   }
}

var pingPong = new Tone.PingPongDelay({ 
  delayTime: "8t",
  feedback: 0.2,
  wet: 0.1}).toDestination()

instruments[1].set(synthJSON);
instruments[1].connect(pingPong)


synthJSON = {
   detune: 0,
   volume: -18,
   oscillator: {
      type: 'square'
   },
   filter: {
      Q: 3,
      type: 'lowpass',
      rolloff: -24,
      frequency: 2000
   },
   envelope: {
      attack: 0.005,
      decay: 0.1,
      sustain: 0.9,
      release: 5
   },
   filterEnvelope: {
      attack: 0.06,
      decay: 1,
      sustain: 0.5,
      release: 2,
      baseFrequency: 200,
      octaves: 2,
      exponent: 2
   }
}

instruments[2].set(synthJSON);
instruments[2].connect(masterReverb)

synthJSON = {
  detune : 0 ,
  volume: -17,
  oscillator : {
    type : 'square'
  },
  filter : {
    Q : 6,
    type : 'lowpass',
    rolloff : -24,
    frequency: 800
    },
  envelope : {
    attack : 0.005 ,
    decay : 0.1 ,
    sustain : 0.9 ,
    release : 1
  },
  filterEnvelope : {
    attack : 0.06 ,
    decay : 0.2 ,
    sustain : 0.5 ,
    release : 2 ,
    baseFrequency : 200 ,
    octaves : 3 ,
    exponent : 2
  }
}

instruments[3].set(synthJSON);
instruments[3].connect(masterReverb)

return instruments
}

let instruments = setInstruments()

let defaultSynth = instruments[0]


// La dernière note jouée sur le clavier du piano
var choosedNote = "C";

// En lui rajoutant l'octave selectionnée on obtient la note (ToneJS utilise ceci comme une fréquence de note)
var choosedNoteWithOctave = ref("C4");


let drums = [];


// Bool pour savoir si le programme a démarré en récupérant les données du morceaux précédent dans le stockage du navigateur
let initiatedByStorage = false


// Variable qui permet de multiplier la durée de note choisie par 1.5 (Note pointée)
let dot = ref(false)
function toggleDot() {
  dot.value ^= true
}

// Quand séléectionné, affiche l'interface mélodie, si le mode song n'est pas actif
let melodyMode = ref(false)
function toggleMelodyMode() {
  melodyMode.value ^= true
  updateSong()
}


// Contient l'ensemble de la mélodie du morceau. Chaque élément de la liste est un chunk de 4 mesures de 16 beat chacune
let melodyPatterns = ref([])

// L'indice de la mesure en cours. Comme les mélodies sont stockées par chunk de 4 mesures l'indice passe de 0 à 4 puis à 8
let melodyPatternsIndice = ref(0)


// Le texte affiché sur le bouton play.
let patternAndMesureLabel = ref("")


let melodyInstruments

function setMelodyInstruments() {
  melodyInstruments = [
  new Tone.DuoSynth(),
  new Tone.PolySynth(
    {voices: 4,
     voice: Tone.FMSynth})
]

let synthJSON = {
   vibratoAmount: 0.4,
   vibratoRate: 5,
   harmonicity: 1.5,
   volume: -6,
   voice0: {
      volume: -12,
      portamento: 0,
      oscillator: {
         type: 'sine'
      },
      filterEnvelope: {
         attack: 0.01,
         decay: 0,
         sustain: 1,
         release: 0.5
      },
      envelope: {
         attack: 0.01,
         decay: 0,
         sustain: 1,
         release: 0.5
      }
   },
   voice1: {
      volume: -10,
      portamento: 0,
      oscillator: {
         type: 'triangle'
      },
      filterEnvelope: {
         attack: 0.01,
         decay: 0,
         sustain: 1,
         release: 0.5
      },
      envelope: {
         attack: 0.01,
         decay: 0,
         sustain: 1,
         release: 0.5
      }
   }
}

melodyInstruments[0].set(synthJSON);
melodyInstruments[0].connect(masterReverb)


synthJSON =  {
   harmonicity: 3,
   modulationIndex: 10,
   detune: 0,
   volume: -15,
   oscillator:{
      type: 'sine'
   },
   envelope:{
      attack: 0.002,
      decay: 0.01,
      sustain: 1,
      release: 0.5
   },
   modulation:{
      type: 'square'
   },
   modulationEnvelope:{
      attack: 0.004,
      decay: 0.005,
      sustain: 0.8,
      release: 0.8
   }
}

var pingPong = new Tone.PingPongDelay({ 
  delayTime: "8t",
  feedback: 0.2,
  wet: 0.1}).toDestination()

  melodyInstruments[1].set(synthJSON);
  melodyInstruments[1].connect(pingPong)

}



function toggleMelodyNote(m, i, j) {
  if (!noteLengthMode.value) {
    // console.log(melodyPatterns.value[0][melodyPatternsIndice.value][i])
      if (melodyPatterns.value[m][melodyPatternsIndice.value][i][j].note === "-") {
        melodyPatterns.value[m][melodyPatternsIndice.value][i][j].note = "?"; // choosedNoteWithOctave.value
      } else {
        melodyPatterns.value[m][melodyPatternsIndice.value][i][j].note = "-";
      }

    updateSong()
  } else {
    setNoteLength(i, j)
  }
  defaultSynth = instruments[i]

}

setMelodyInstruments()

let songStart = ref(0)
function setSongStart(i) {
  if (songStart.value != i) {
    songStart.value = i
  } else {
    songStart.value = 0
  }
  updateSong()
}

let urlParams = new URLSearchParams(window.location.search)

if (urlParams.has('data')) {
  let compressedData = urlParams.get('data')

  let data = JSON.parse(decompressFromEncodedURIComponent(compressedData))

  voicesPatterns.value = data.voices
  melodyPatterns.value = data.melody
  drumsPatterns = data.drums
  bpm.value = data.bpm
  song.value = data.song
} else if ((storage.has('voices') === true) && (storage.has('drums') === true) && (storage.has('song') === true) && (storage.has('bpm') === true) && (storage.has('melody') === true)) {

  voicesPatterns.value = storage.get('voices')
  drumsPatterns.value = storage.get('drums')
  song.value = storage.get('song')
  bpm.value = storage.get('bpm')
  melodyPatterns.value = storage.get('melody')
  initiatedByStorage = true
  updateSong()
} else {
  resetPaterns()
}

function songSetup() {
  if (songSetupEnCours.value===true) {
    songSetupEnCours.value = false
    if (song.value.length === 0) {
      if (isPlaying === true) {
        Tone.getTransport().stop();
        isPlaying = false;
        playStatus.value = "Play"
        beat.value = 0;
      }
      song.value.push(0)
    }
  } else {
    songSetupEnCours.value = true
  }
}

function songAddPattern(i) {
  if (song.value.length < 42) {
  song.value.push(i)
  updateSong()

  }
}

let clave = new Tone.Player({
      onload: () => resolve(clave),
      url: '/asset__clave.opus',
      volume: -6
    }).connect(masterReverb)

onMounted(() => {
  dialog_reset_patterns.close()
  let samples = [
    [{name: "/asset__ride.opus", volume: -18},{name: "/asset__crash.opus", volume: -15}, {name: "/asset__crash_2.opus", volume: -15}],
    [{name: "/asset__hh_closed.opus", volume: -15},{name: "/asset__hh_open.opus", volume: -15}, {name: "/asset__hh_open_2.opus", volume: -15}],
    [{name: "/asset__snare.opus", volume: -10},{name: "/asset__snare_2.opus", volume: -8}, {name: "/asset__clap.opus", volume: -8}],
    [{name: "/asset__kick.opus", volume: -6},{name: "/asset__tom_low.opus", volume: -8}, {name: "/asset__tom_mid.opus", volume: -8}],
  ]

  for (let i = 0; i < samples.length; i++) {
    drums.push([])
    for (let j = 0; j < samples[i].length; j++) {
    drums[i].push(new Tone.Player({
      onload: () => resolve(drums[i][j]),
      url: samples[i][j].name,
      volume: samples[i][j].volume
    }).connect(masterReverb))
  }
}
});

function saveUrl() {
  let saveData = {
    voices: voicesPatterns.value,
    drums: drumsPatterns.value,
    song: song.value,
    bpm: bpm.value,
    melody: melodyPatterns.value
  }
  // let compressedData = JSON.stringify(compress(data))
  // let url = "http://localhost:5173?data=" + encodeURIComponent(compressedData)
  window.navigator.clipboard.writeText(compressToEncodedURIComponent(JSON.stringify(saveData)))


}




function copy(mode) {
  if (mode === 'drums') {
    var data = JSON.parse(JSON.stringify({value: drumsPatterns.value[visiblePattern.value]}))
    clipboard.value = { 'type': 'Drums', 'data': data.value}
  } else if (mode === 'voices') {
    var data = JSON.parse(JSON.stringify({value: voicesPatterns.value[visiblePattern.value]}))
    clipboard.value = { 'type': 'Voices', 'data': data.value}
  }

}


function paste() {
  if (clipboard.value.type === 'Drums') {
    var data = JSON.parse(JSON.stringify({value: clipboard.value.data}))
    drumsPatterns.value[visiblePattern.value] = data.value
  } else if (clipboard.value.type === 'Voices')
  {
    var data = JSON.parse(JSON.stringify({value: clipboard.value.data}))
    voicesPatterns.value[visiblePattern.value] = data.value
  }
  updateSong();
}


function toggleNote(i, j) {
  if (!noteLengthMode.value) {

  if (voicesPatterns.value[visiblePattern.value][i][j].note === "-") {
    voicesPatterns.value[visiblePattern.value][i][j].note = "?"; // choosedNoteWithOctave.value
  } else {
    voicesPatterns.value[visiblePattern.value][i][j].note = "-";
  }
  updateSong();
} else {
  setNoteLength(i, j)
}
  defaultSynth = instruments[i]
}


function toggleDrumNote(i, j) {
  if (drumsPatterns.value[visiblePattern.value][i][j] === 1) {
    drumsPatterns.value[visiblePattern.value][i][j] = 2
  } else if (drumsPatterns.value[visiblePattern.value][i][j] === 0) {
    drumsPatterns.value[visiblePattern.value][i][j] = 1
  } else if (drumsPatterns.value[visiblePattern.value][i][j] === "-") {
      drumsPatterns.value[visiblePattern.value][i][j] = 0
  } else {
    drumsPatterns.value[visiblePattern.value][i][j] = "-"
  }
  updateSong();
}



function toggleNoteLenghtMode() {
  noteLengthMode.value ^= true
}

let noteLengthMode = ref(false)
function setNoteLength(i, j) {

  let ajoutDot = ""

  if (dot.value) {
    ajoutDot = "."
  }
  if (melodyMode.value) {
    melodyPatterns.value[0][melodyPatternsIndice.value][i][j].duree = possibleRythmeTranslation[choosedRythm.value] + ajoutDot
  
  }
    
  else {
  voicesPatterns.value[visiblePattern.value][i][j].duree = possibleRythmeTranslation[choosedRythm.value] + ajoutDot
}


}

function setNote(note) {
  choosedNoteWithOctave.value = note + octave.value

  let ajoutDot = ""

  if (dot.value) {
    ajoutDot = "."
  }

  if (!soundDuringPlay.value) {

    if (!melodyMode.value) {
    for (let i = 0; i < instruments.length; i++) {
      for (let j = 0; j < voicesPatterns.value[visiblePattern.value][i].length; j++) {
        if (voicesPatterns.value[visiblePattern.value][i][j].note === "?") {
        
          voicesPatterns.value[visiblePattern.value][i][j] = {note: choosedNoteWithOctave.value, duree: possibleRythmeTranslation[choosedRythm.value] + ajoutDot}
          defaultSynth.triggerAttackRelease(choosedNoteWithOctave.value, "8n")
          updateSong()
          return
        }
      }
    }
  } else {
    for (let i = 0; i < melodyInstruments.length; i++) {
      for (let j = 0; j < melodyPatterns.value[i][melodyPatternsIndice.value].length; j++) {

        for (let k = 0; k < patternsNumber; k++) {
        if (melodyPatterns.value[i][melodyPatternsIndice.value][j][k].note === "?") {
        
          melodyPatterns.value[i][melodyPatternsIndice.value][j][k] = {note: choosedNoteWithOctave.value, duree: possibleRythmeTranslation[choosedRythm.value] + ajoutDot}
          // defaultSynth.triggerAttackRelease(choosedNoteWithOctave.value, "8n")
          // console.log(melodyPatterns.value[i][melodyPatternsIndice.value][j][k])
          updateSong()
          return
        }
        }
      }
    }
  }
  
    if (!isPlaying) {
      defaultSynth.triggerAttackRelease(choosedNoteWithOctave.value, "8n")
    }
  } else {
    defaultSynth.triggerAttackRelease(choosedNoteWithOctave.value, "8n")
  }
}


function removeLastSongPattern() {
  song.value.pop()
}


// Met à jour l'interface
function updateSong() {
  completeSong = []

  let local_song = []
  let local_song_start = 0

  if (singlePatternPlay.value) {
    local_song.push(visiblePattern.value)
  } else {
    local_song = song.value
    local_song_start = songStart.value
  }

  if (melodyMode.value) {
    patternAndMesureLabel.value = "Mes:" + (melodyPatternsIndice.value *4 +1)
  } else {
    patternAndMesureLabel.value = "Pat:" + (visiblePattern.value +1)
  }


  // let visibleMelodyPattern.value = []



  for (
    let instrumentIndex = 0;
    instrumentIndex < instruments.length;
    instrumentIndex++
  ) {
    for (let patternIndex = local_song_start; patternIndex < local_song.length; patternIndex++) {
      completeSong.push([])
      for (let noteIndex = 0; noteIndex < 16; noteIndex++) {
        completeSong[instrumentIndex].push(
          voicesPatterns.value[local_song[patternIndex]][instrumentIndex][noteIndex]
        );
      }
    }
  }



  completeDrumsSong = []

  for (let drumIndex = 0; drumIndex < drums.length; drumIndex++) {
    for (let patternIndex = local_song_start; patternIndex < local_song.length; patternIndex++) {
      completeDrumsSong.push([])
      for (let noteIndex = 0; noteIndex < 16; noteIndex++) {
        completeDrumsSong[drumIndex].push(
          drumsPatterns.value[local_song[patternIndex]][drumIndex][noteIndex]
      );
    }
  }
}
}



function changeOctave(mode) {
  if (mode === "up") {
    octave.value = Math.min(octave.value + 1, 7);
  } else {
    octave.value = Math.max(octave.value - 1, 1);
  }
  choosedNoteWithOctave.value = choosedNote + octave.value;
}


function changePattern(mode) {
  if (melodyMode.value) {
    if (mode === "next")
  {
    melodyPatternsIndice.value = Math.min(melodyPatternsIndice.value + 1, 128-1)
  } else{
    melodyPatternsIndice.value = Math.max(melodyPatternsIndice.value - 1, 0)
  }
  } else {
  if (mode === "next") {
    visiblePattern.value = Math.min(visiblePattern.value + 1, patternsNumber-1);
  } else {
    visiblePattern.value = Math.max(visiblePattern.value - 1, 0);
  }
}
  updateSong()
}

// Boucle principale
function play() {
  if (audioContextActivated === false) {
    startAudioContext();
  }

  if (isPlaying === true) {
    Tone.getTransport().stop();
    isPlaying = false;
    playStatus.value = "Play"
    beat.value = 0;
    storage.set('melody', melodyPatterns.value)
    storage.set('voices', voicesPatterns.value)
    storage.set('drums', drumsPatterns.value)
    storage.set('song', song.value)
    storage.set('bpm', bpm.value)

  } else {
    if (transportCreated === false) {
      if (initiatedByStorage===true) {
        updateSong()
      }
      Tone.getTransport().scheduleRepeat((time) => {
        for (
          let instrumentIndex = 0;
          instrumentIndex < instruments.length;
          instrumentIndex++
        ) {
          if (!["-"].includes(completeSong[instrumentIndex][beat.value].note)) {
            if (["?"].includes(completeSong[instrumentIndex][beat.value].note)) {
              clave.start(time)
            } else {
            let note = completeSong[instrumentIndex][beat.value].note
            let duree = completeSong[instrumentIndex][beat.value].duree

            let humanTime = time - 0.005 + (Math.floor(Math.random() * 10))/1000
            instruments[instrumentIndex].triggerAttackRelease(
              note,
              duree,
              humanTime
            );
          }
        }
      }

        for (let i = 0; i < drums.length; i++) {
          if ([0, 1, 2].includes(completeDrumsSong[i][beat.value])) {
            drums[i][completeDrumsSong[i][beat.value]].start(time)
          } 
        }
        let local_beat = beat.value
        local_beat += songStart.value * 16
        const index = Math.floor(local_beat / 64); // 64 items per sub-array, so divide by 64 to get the main array index
        local_beat -= index * 64; // Adjust beat for further calculations within the selected sub-array

        const subIndex = Math.floor(local_beat / 16); // Each sub-array has 16 items, so divide by 16 to get the sub-array index
        local_beat -= subIndex * 16; // Adjust beat for further calculations within the selected array


        for (let i = 0; i < melodyInstruments.length; i++) {
          if (!["-"].includes(melodyPatterns.value[i][index][subIndex][local_beat].note)) {
            if (["?"].includes(melodyPatterns.value[i][index][subIndex][local_beat].note)) {
              clave.start(time)
            } else {
              let note = melodyPatterns.value[i][index][subIndex][local_beat].note
              let duree = melodyPatterns.value[i][index][subIndex][local_beat].duree
              let humanTime = time - 0.005 + (Math.floor(Math.random() * 10))/1000
              // console.log(note)

              melodyInstruments[i].triggerAttackRelease(
              note,
              duree,
              humanTime
            );
            }
            
        }
      }

        let local_song = []

        if (singlePatternPlay.value) {
          local_song.push(visiblePattern.value)
        } else {
        local_song = JSON.parse(JSON.stringify(song.value))

        if (songStart.value != 0) {
          for (let i = 0; i < songStart.value; i++) {
            local_song.shift()
          }
        }
      }

        if (isPlaying === true) {
          // console.log(beat.value)
          beat.value = (beat.value + 1) % (local_song.length * 16);
          if (beat.value===1) {
            currentSongPattern.value = local_song[0]
          }          
          if (beat.value%16===0) {
            currentSongPattern.value = local_song[beat.value/16]
          }
          if (followHead.value) {
            if (melodyMode.value) {
              melodyPatternsIndice.value =  Math.floor(beat.value / 64)
            }
            else {
              visiblePattern.value =  currentSongPattern.value
            }
            updateSong()
          }
        }
      }, "16n");
      Tone.getTransport().bpm.value = bpm.value
      transportCreated = true;
    }
    Tone.getTransport().start();
    isPlaying = true;
    playStatus.value = "Pause"
  }
}


function exportSong() {
  let exportType =  exportFromJSON.types.txt
  let fileName = 'song.json'

  let data = compressToBase64(JSON.stringify({voices: voicesPatterns.value,
          drums: drumsPatterns.value,
          song: song.value,
          bpm: bpm.value,
          melody: melodyPatterns.value
  }))

  exportFromJSON({ data, fileName, exportType })

}


</script>








<template>
  <DialogsWrapper />
  <div class="fullscreen-wrapper">
    <div class="grid-container0">
      <button @click="play" class="play-button">
        {{ bpm }} | <span class="rythmes"> {{ rythmes[choosedRythm] }} </span>|
        {{ patternAndMesureLabel }} | {{ playStatus }}
      </button>
      <!-- <div class="grid-container1" v-if="songSetupEnCours === true">
        <div class="grid-container2"></div>
      </div> -->
      <div class="grid-container1">
        <div class="grid-container2">
          <div v-if="songSetupEnCours === true">
            <div class="note-row">
              <div
                v-for="i in 16"
                v-bind:key="'patterntochoose' + i"
                class="note"
                :class="{ on_beat: i % 4 === 0 }"
                @click="songAddPattern(i - 1)"
                style="padding: 1.5em"
                :num="i"
              ></div>
              <div class="grid-container1">
                <div>
                  <p
                    style="display: inline"
                    v-for="(pattern, i) in song"
                    v-bind:key="'songPattern' + pattern"
                    @click="setSongStart(i)"
                  >
                    <span
                      :style="[
                        i === songStart ? 'color: red' : '',
                        i * 16 > beat - 15 + songStart * 16 &&
                        i * 16 < beat + songStart * 16
                          ? 'color: orange'
                          : '',
                      ]"
                    >
                      {{ pattern + 1 }}
                    </span>
                    <span v-if="i + 1 < song.length">, </span>
                  </p>
                </div>
                <button @click="removeLastSongPattern()" class="undo-button">
                  Undo
                </button>
              </div>
            </div>
          </div>
          <div v-if="songSetupEnCours === false && melodyMode">
            <div
              v-for="(pattern, i) in melodyPatterns[0][melodyPatternsIndice]"
              class="note-row"
              v-bind:key="'melody_' + pattern + i"
            >
              <note
                v-for="(note, j) in pattern"
                :num="note.note"
                @click="toggleMelodyNote(0, i, j)"
                v-bind:key="'melody_' + note.note + j"
                :class="{
                  played: note.note != '-',
                  on_beat: j % 4 === 0,
                  playhead: beat === melodyPatternsIndice * 4 * 16 + j + 16 * i,
                }"
              >
              </note>
            </div>
          </div>

          <div class="gap"></div>
          <div
            v-for="(pattern, i) in melodyPatterns[1][melodyPatternsIndice]"
            class="note-row"
            v-bind:key="'melody2_' + pattern + i"
          ></div>

          <div v-if="songSetupEnCours === false && !melodyMode">
            <div
              v-for="(pattern, i) in voicesPatterns[visiblePattern]"
              class="note-row"
              v-bind:key="pattern + i"
            >
              <note
                v-for="(note, j) in pattern"
                :num="note.note"
                @click="toggleNote(i, j)"
                :class="{
                  played: note.note != '-',
                  on_beat: j % 4 === 0,
                  playhead:
                    j === beat % 16 && visiblePattern === currentSongPattern,
                }"
                v-bind:key="note.note + j"
              >
              </note>
            </div>

            <div class="gap"></div>

            <div
              v-for="(pattern, i) in drumsPatterns[visiblePattern]"
              class="note-row"
              v-bind:key="'d' + pattern + i"
            >
              <note
                v-for="(note, j) in pattern"
                :num="note"
                @click="toggleDrumNote(i, j)"
                :class="{
                  played: note != '-',
                  on_beat: j % 4 === 0,
                  playhead:
                    j === beat % 16 && visiblePattern === currentSongPattern,
                }"
                v-bind:key="'d' + note + i"
              >
              </note>
            </div>
          </div>
        </div>

        <div class="grid-container3">
          <div class="grid-container4">
            <button @click="changePattern('prev')" class="oct-button">
              Pat -
            </button>
            <button @click="changePattern('next')" class="oct-button">
              Pat +
            </button>
            <button @click="changeOctave('down')" class="oct-button">
              Oct {{ octave - 1 }}
            </button>
            <button @click="changeOctave('up')" class="oct-button">
              Oct {{ octave + 1 }}
            </button>
          </div>
          <piano v-on:noteSelected="setNote"></piano>
        </div>
      </div>

      <div class="grid-container6">
        <div class="grid-container5">
          <button @click="fullscreen.toggle()" class="function-button">
            Full Screen
          </button>
          <button
            @click="toggleFollow()"
            class="function-button"
            :style="followHead ? 'background: #b46143;' : ''"
          >
            follow
          </button>
          <button
            @click="toggleSoundDuringPlay()"
            class="function-button"
            :style="soundDuringPlay ? 'background: #b46143;' : ''"
          >
            Live Snd
          </button>
          <button @click="askResetPaterns()" class="function-button">
            Reset
          </button>
          <button @click="exportSong()" class="function-button">Export</button>
        </div>
        <div class="grid-container5-1">
          <button
            @click="toggleSinglePatternPlay()"
            class="function-button"
            :style="singlePatternPlay ? 'background: #b46143;' : ''"
          >
            Single
          </button>

          <button @click="changeRythm('up')" class="function-button">
            Rythm +
          </button>
          <button
            @click="toggleMelodyMode()"
            class="function-button"
            :style="
              melodyMode && !songSetupEnCours ? 'background: #b46143;' : ''
            "
          >
            Mélodie
          </button>
          <button @click="saveUrl()" class="function-button">URL</button>
          <button @click="setBpm('up')" class="function-button">BPM +</button>
        </div>
        <div class="grid-container5">
          <button @click="copy('drums')" class="function-button">
            Copy Drum
          </button>
          <button @click="copy('voices')" class="function-button">
            Copy Voices
          </button>
          <button @click="paste()" class="function-button">
            Paste {{ clipboard.type }}
          </button>
          <button
            @click="songSetup()"
            class="function-button"
            :style="songSetupEnCours ? 'background: #b46143;' : ''"
          >
            Song
          </button>
          <button @click="open()" class="function-button">Import</button>
        </div>
        <div class="grid-container5-1">
          <button
            @click="toggleNoteLenghtMode()"
            class="function-button"
            :style="noteLengthMode ? 'background: #b46143;' : ''"
          >
            Set Length
          </button>

          <button @click="changeRythm('down')" class="function-button">
            Rythm -
          </button>
          <button
            @click="toggleDot()"
            class="function-button"
            :style="dot ? 'background: #b46143;' : ''"
          >
            .
          </button>
          <button class="function-button">-</button>
          <button @click="setBpm('down')" class="function-button">BPM -</button>
        </div>
      </div>
    </div>
  </div>
</template>



<style>
@font-face {
  font-family: "rythmes";
  src: url("/asset__rythmes.ttf");
}

span.rythmes {
  font-family: "rythmes";
}

html {
  background: #56626a;
}
.gap {
  padding-top: 1em;
}

.play-button {
  width: 20%;
  margin-left: 5em;
  align-items: left;
}
.undo-button {
  margin-left: 0.5em;
  margin-right: 0.9em;
}

.oct-button {
  margin-left: 2em;
  margin-right: 2em;
  margin-bottom: 0.2em;
}

.function-button {
  margin-left: 0.5em;
  margin-right: 0.5em;
  margin-bottom: 0.2em;
  height: 3.5em;
  width: 7em;
  font-size: xx-small;
}

.grid-container0 {
  display: grid;
  grid-template-columns: 770px;
  height: 2em;
}

.grid-container1 {
  display: grid;
  grid-template-columns: auto auto;
}

.grid-container2 {
  padding-top: 2em;
  display: grid;
  grid-template-columns: auto;
}

.grid-container3 {
  display: grid;
  grid-template-columns: auto;
}

.grid-container4 {
  display: grid;
  grid-template-columns: auto auto;
  margin-left: 2em;
}
.grid-container5 {
  display: grid;
  grid-template-columns: auto auto auto auto auto;
  margin-right: 2em;
}

.grid-container5-1 {
  display: grid;
  grid-template-columns: auto auto auto auto auto;
  margin-left: 2em;
}

.grid-container6 {
  display: grid;
  grid-template-columns: 50% 50%;
  grid-template-rows: auto auto;
  margin-top: 0.6em;
}

.note-row {
  display: block;
  margin: 0;
  position: relative;
  left: 0em;
}
</style>
