<template>
    <div>
      <video style="display:block" ref="video" width="640" height="480" autoplay></video>
      <button @click="toggleRecording">{{ isRecording ? 'Stop' : 'Start' }}</button>
      <p>{{ result }}</p>
    </div>
</template>
  
<script>
  export default {
    data() {
      return {
        recordedBlobs: [],
        mediaRecorder: null,
        isRecording: false,
        result: '',
      };
    },
    mounted() {
      navigator.mediaDevices.getUserMedia({ video: true, audio: false })
        .then((stream) => {
          this.mediaRecorder = new MediaRecorder(stream);
          this.mediaRecorder.onstop = this.onStop;
          this.mediaRecorder.ondataavailable = this.onDataAvailable;
          this.$refs.video.srcObject = stream;
        });
    },
    methods: {
      toggleRecording() {
        if (this.isRecording) {
          this.mediaRecorder.stop();
          this.result = 'waiting...';
        } else {
          this.mediaRecorder.start(10);
          this.recordedBlobs = [];
          this.result = '';
        }
        this.isRecording = !this.isRecording;
      },
      onDataAvailable(event) {
        if (event.data && event.data.size > 0) {
          this.recordedBlobs.push(event.data);
        }
      },
      onStop(event) {
        const lastChunk = event.data;
        if (lastChunk && lastChunk.size > 0) {
          this.recordedBlobs.push(lastChunk);
        }
        const blob = new Blob(this.recordedBlobs, { type: 'video/webm' });
        const reader = new FileReader();
        reader.readAsDataURL(blob);
        reader.onloadend = () => {
          const base64data = reader.result;
          fetch('https://api.sign-speak-dev.com/recognition', {
            method: 'POST',
            headers: {
              'x-api-key': 'API_KEY',
              'Content-Type': 'text/plain'
            },
            body: base64data.split(',')[1]
          })
            .then(response => response.text())
            .then(data => {
              this.result = data;
              this.$emit('resultReceived', data);
            })
            .catch((error) => {
              console.log('AJAX error: ', error);
            });
        };
      },
    },
  };
</script>