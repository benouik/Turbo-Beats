
<script setup lang="js">
defineProps(['num'])

</script>

<template>
    <div class="note"><div class="note-name" :class="{ 'hidden': num==='-'}">{{ num }}</div></div>
</template>


<style>
.note-name {
    display: flex;
    justify-content: center;
    align-items: center;
    font-size: 0.75em;
    line-height: 2em;
    text-align: center;
    /* display: inline-block; */
 
  }

  .note {
    width: 1.6em;
    height: 1.57em;
    display: inline-block;
    border-top: 1px solid #2d3f4d;
    border-bottom: 1px solid #2d3f4d;
    border-left: 1px solid #2d3f4d;
    border-right: 1px solid #2d3f4d;
    border-radius: 7px;
    background: #335346;
    margin-left: 1px;
    /* padding-left: 2px; */
  }

  /* .note-name {
    width:50px;
    height:50px;
    display: inline-block;
    padding: 10px;
    text-align: center;

  } */
  .note.on_beat {
    border-top: 1px solid #8ba8a6;
    border-bottom: 1px solid #8ba8a6;
    border-left: 1px solid #8ba8a6;
    border-right: 1px solid #8ba8a6;
  }
  .note.playhead {
    border-top: 1px solid #c42828;
    border-bottom: 1px solid #851919;
    border-left: 1px solid #a11414;
    border-right: 1px solid #c42828;

    /* border-top: 1px solid #14130e;
    border-bottom: 1px solid #14130e;
    border-left: 1px solid #14130e;
    border-right: 1px solid #14130e; */
  }

  .note.played {
    background: #b46143;
  }

  .note-name.hidden {
   color: #2d3f4d; 
  }
  </style>