<script lang="ts">
  import { onMount } from 'svelte';

  let recording = false;
  let mediaRecorder: MediaRecorder;
  let audioChunks: Blob[] = [];
  let audioBlob: Blob | null = null;
  let timer = 0;
  let timerInterval: NodeJS.Timer;
  let transcript = '';
  let name = '';
  let nameLocked = false;
  let buttonLock=true;
  let job_position ='Account Manager';
  let syarat = 'Berpengalaman 2 tahun di B2B sales';
  let question ="Hi, salam kenal. Boleh perkenalan dan menceritakan pengalaman anda yang berkaitan dengan "+job_position+"?";

  let recognition: SpeechRecognition;
  const SpeechRecognition = window.SpeechRecognition || (window as any).webkitSpeechRecognition;

  onMount(() => {
    if (SpeechRecognition) {
      recognition = new SpeechRecognition();
      recognition.continuous = true;
      recognition.interimResults = true;
      recognition.lang = 'id-ID';

      recognition.onresult = (event: SpeechRecognitionEvent) => {
        const result = Array.from(event.results)
          .map(res => res[0].transcript)
          .join('');
        transcript = result;
      };
      // speakQuestion(question);
      recognition.onerror = e => console.error('Recognition error:', e);
    }
  });

  async function startRecording() {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    mediaRecorder = new MediaRecorder(stream);

    mediaRecorder.ondataavailable = event => {
      audioChunks.push(event.data);
    };

    mediaRecorder.onstop = () => {
      audioBlob = new Blob(audioChunks, { type: 'audio/webm' });
    };

    audioChunks = [];
    mediaRecorder.start();
    if (recognition) recognition.start();
    timer = 0;
    timerInterval = setInterval(() => timer++, 1000);
    recording = true;
  }

  function stopRecording() {
    mediaRecorder.stop();
    if (recognition) recognition.stop();
    clearInterval(timerInterval);
    recording = false;
    buttonLock=false;
  }

  async function sendAudio() {
    if (!audioBlob) return;

    buttonLock=true;

    if (!name || name.trim() === '') {
      alert('Nama harus diisi sebelum mengirim.');
      return;
    }

    const formData = new FormData();
    formData.append('name', name);
    // formData.append('audio', audioBlob, 'interview.webm');
    formData.append('timestamp', new Date().toISOString());
    formData.append('question', question);
    formData.append('transcript', transcript);
    formData.append('syarat', syarat);
    formData.append('job_position', job_position);

    const res=await fetch('https://n8n.yesvara.com/webhook/interview-submit', {
      method: 'POST',
      body: formData
    });

    nameLocked = true;
    alert('Jawaban berhasil dikirim');

    const data = await res.json();

    // ✅ Update pertanyaan berikutnya
    const nextQuestion = data[0]?.next_question;

    if (nextQuestion) {
      question = nextQuestion; 
      buttonLock=true;
      console.log("response:", data);
      console.log("next_question:", data[0]?.next_question);
      transcript='';
    }

    
  }

  function speakQuestion(text: string) {
    const utterance = new SpeechSynthesisUtterance(text);
    utterance.lang = 'id-ID'; // Bahasa Indonesia
    utterance.rate = 1.3;       // Kecepatan bicara (0.1 – 10)
    speechSynthesis.speak(utterance);
  }

 
</script>

<svelte:head>
  <link
    rel="stylesheet"
    href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.0/css/all.min.css"
    integrity="sha512-..."
    crossorigin="anonymous"
    referrerpolicy="no-referrer"
  />
</svelte:head>

<style>
  :global(body) {
    margin: 0;
    font-family: 'Segoe UI', sans-serif;
    background-color: #01212e;
    color: white;
  }

  .container {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 1rem;
    text-align: center;
    justify-content: center;
  }

  .logo {
    max-width: 75px;
    margin-bottom: 2rem;
  }

  .question {
    font-size: 1.5rem;
    font-size:20px;
    margin-bottom: 2rem;
    max-width: 600px;
  }

  .controls {
    display: flex;
    gap: 1rem;
    flex-wrap: wrap;
    justify-content: center;
  }

  button {
    background-color: #01a3a4;
    border: none;
    padding: 1rem 2rem;
    color: white;
    font-size: 1rem;
    width: 130px;
    border-radius: 8px;
    cursor: pointer;
    transition: background 0.3s ease;
    display: flex;
    align-items: center;   
    gap: 0.5rem;
  }

  button:hover {
    background-color: #00cec9;
  }

  button:disabled {
    background-color: #555;
    cursor: not-allowed;
  }

  input[disabled] {
    background-color: #ffffff10;
    color: white;
    cursor: not-allowed;
  }

  .timer {
    margin-top: 1rem;
    font-size: 1.2rem;
  }

  .transcript {
    /* margin-top: 2rem; */
    background: #ffffff10;
    padding: 1rem;
    border: 1px solid #ffffff30;
    border-radius: 10px;
    max-width: 600px;
    margin-bottom:15px;
  }
</style>

<div class="container">
  <img src="/logo.png" alt="Logo" class="logo" />

  <div class="question">    
    {question}
  </div>

{#if recording}
    <div class="timer">Durasi: {timer} detik</div>
  {/if}

  {#if transcript}
    <div class="transcript" >
      <strong>Jawaban anda:</strong><br />
      {transcript}
    </div>
  {/if}

  <div class="input-nama" style="margin-bottom: 1.5rem;">
    <label for="name">Peserta:</label><br />
    <input
      id="name"
      type="text"
      bind:value={name}
      required
      disabled={nameLocked}
      style="padding: 0.5rem; border-radius: 8px; border: none; width: 250px;"
      placeholder="Tuliskan nama Anda disini"
    />
  </div>

  <div class="controls">
    {#if !recording}
      <button on:click={startRecording}>
        <i class="fas fa-microphone"></i> Jawab
      </button>
    {:else}
      <button on:click={stopRecording} style="background-color:red">
        <i class="fas fa-stop"></i> Cukup
      </button>
    {/if}
    <button on:click={sendAudio} disabled={buttonLock}>
      <i class="fas fa-paper-plane"></i> Kirim
    </button>
  </div>

  
</div>