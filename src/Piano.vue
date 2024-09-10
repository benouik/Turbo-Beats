<script setup lang="js">

    const notes = ["C", "C#", "D", "D#", "E","S#", "F", "F#", "G", "G#", "A", "A#", "B"]
    const emit = defineEmits(['noteSelected'])
    function selectNote(note) {
        emit('noteSelected', note)
    }
</script>


<template>
    <div class="piano-container">
        <ul class="piano-keys-list">
            <!-- <li class="piano-keys white-key" data-key="01"></li>
          <li class="piano-keys black-key" data-key="02"></li>
          <li class="piano-keys white-key" data-key="03"></li>
          <li class="piano-keys black-key" data-key="04"></li> -->
            <li v-for="note in notes" class="piano-keys"
                :class="{ 'black-key': note.includes('#'), 'white-key': !note.includes('#'), 'hidden': note.includes('S')}"
                @click="selectNote(note)" v-bind:key="note">
            </li>
        </ul>
    </div>
</template>


<style>
.piano-container {
    width: 100%;
    display: flex;
    align-items: center;
    justify-content: flex-start;
    overflow: auto;
}

.piano-keys-list {
    list-style: none;
    display: flex;
    justify-content: center;
    /* position: absolute;
    left: 470px;
    top: 140px; */
}

.piano-keys {
    width: 3em;
    cursor: pointer;
    position: relative;
    height: 10rem;
    border-radius: 10px;
    border: 1px solid #000;
    list-style-type: none
}

.white-key {
    background-color: #fff;
    margin: 0 0.3em 0 0.3em;
}

.black-key {
    width: 2em;
    height: 6rem;
    border-radius: 5px;
    border-top-left-radius: 0;
    border-top-right-radius: 0;
    background-color: #000;
    z-index: 2;
    margin: 0 -1.35em 0 -1.35em;
}
.hidden {
    visibility: hidden;
}
</style>