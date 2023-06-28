<script setup>
import { reactive, ref } from 'vue'

const formSize = ref('default')
const ruleFormRef = ref()
const ruleForm = reactive({
  firstname: '',
  lastname: '',
  email: '',
  age: undefined,
  adresse: '',
  ville: undefined,
  parrainage: undefined
})

const rules = reactive({
  firstname: [
    { required: true, message: 'Merci de renseigner votre Prénom', trigger: 'blur' }
  ],
  lastname: [
    { required: true, message: 'Merci de renseigner votre Nom', trigger: 'blur' }
  ],
  email: [
    { required: true, message: 'Merci de renseigner votre Email', trigger: 'blur' }
  ],
  age: [
    { required: true, message: 'Merci de renseigner votre Age', trigger: 'blur' },
    { validator: (message, value) => value < 18 || value > 150 ? false : true, message: 'Merci de renseigner un age entre 18 et 150', trigger: 'blur' }
  ],
  ville: [
    {
      required: true,
      message: 'Merci de renseigner votre Ville',
      trigger: 'change',
    },
  ]
})
</script>

<template>
  <div class="logo">
    <img src="./assets/logo.png" alt="logo">
  </div>
  <el-form ref="ruleFormRef" label-position="top" :model="ruleForm" :rules="rules" label-width="120px"
    class="demo-ruleForm" :size="formSize" status-icon>
    <el-form-item label="Prénom" prop="firstname">
      <el-input v-model="ruleForm.firstname" />
    </el-form-item>
    <el-form-item label="Nom" prop="lastname">
      <el-input v-model="ruleForm.lastname" />
    </el-form-item>
    <el-form-item label="Email" prop="email">
      <el-input type="email" placeholder="test@example.com" v-model="ruleForm.email" />
    </el-form-item>
    <el-form-item label="Age" prop="age">
      <el-input min="18" max="150" type="number" v-model="ruleForm.age" />
    </el-form-item>
    <el-form-item label="Adresse" prop="adresse">
      <el-input type="string" v-model="ruleForm.adresse" />
    </el-form-item>
    <el-form-item label="Ville" prop="ville">
      <el-select v-model="ruleForm.ville" placeholder="Choissir une ville">
        <el-option v-for="item in Ville" :label="item.nom" :value="item.idVille" />
      </el-select>
    </el-form-item>
    <el-form-item label="Parrainage" prop="parrainage">
      <el-select-v2 v-model="ruleForm.parrainage" filterable :options="options" placeholder="Parrains" />
    </el-form-item>
    <el-form-item>
      <el-button type="primary" @click="submitForm(ruleFormRef, ruleForm)">
        Envoyer
      </el-button>
      <el-button @click="resetForm(ruleFormRef)">Reset</el-button>
      <el-button @click="requestCameraAccess()">Prendre une photo</el-button>
    </el-form-item>
  </el-form>
  <video id="video-element"></video>
  <canvas id="canvas-element" style="width: 100%;margin-bottom: 2rem;"></canvas>
</template>

<script>
let showCam = false;
let photo = '';

const { data: { Ville } } = await fetch("http://152.228.217.28:5000/api/graphql/v1/", {
  method: "POST",
  headers: {
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    query: `
      {
        Ville {
          idVille
          nom
          coordonnees
        }
      }
      `
  }),
})
  .then(res => res.json());

const { data: { Visiteur } } = await fetch("http://152.228.217.28:5000/api/graphql/v1/", {
  method: "POST",
  headers: {
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    query: `
      {
        Visiteur {
          idVisiteur
          nom
          prenom
          email
          age
          adresse
          idParrain
          idVille
        }
      }
      `
  }),
})
  .then(res => res.json());

Notification.requestPermission();

const options = reactive(Array.from(Visiteur).map((parrain) => ({
  value: `${parrain.idVisiteur}`,
  label: `${parrain.nom} ${parrain.prenom}`,
})))

async function requestCameraAccess() {
  try {
    const videoElement = document.getElementById('video-element');
    if (showCam === false || videoElement.srcObject === null) {
      const stream = await navigator.mediaDevices.getUserMedia({ video: true });
      videoElement.srcObject = stream;
      videoElement.play();
      showCam = true;
    } else if (showCam === true) {
      const canvasElement = document.getElementById('canvas-element');
      capturePicture(videoElement, canvasElement);
    }
  } catch (error) {
    console.error('Error accessing camera:', error);
  }
}

function capturePicture(videoElement, canvasElement) {
  const canvasContext = canvasElement.getContext('2d');
  canvasContext.drawImage(videoElement, 0, 0, canvasElement.width, canvasElement.height);

  videoElement.pause();
  videoElement.srcObject = null;

  const dataURL = canvasElement.toDataURL('image/png');

  // const downloadLink = document.createElement('a');
  // downloadLink.href = dataURL;
  // downloadLink.download = 'captured-image.png';

  // downloadLink.click();

  createFileWithDataURL(dataURL);
}

function createFileWithDataURL(dataURL) {
  photo = dataURL.replace(/^data:image\/png;base64,/, '');
}

const submitForm = async (formEl, data) => {
  if (!formEl) return
  let perm = Notification.permission;
  if (perm === 'denied') {
    return
  } else if (perm === 'granted') {
    await formEl.validate(async (valid, fields) => {
      if (valid) {
        console.log(photo);
        await fetch("http://152.228.217.28:5000/api/graphql/v1/", {
          method: "POST",
          headers: {
            "Content-Type": "application/json"
          },
          body: JSON.stringify({
            query: `mutation { VisiteurCreate(Visiteur: {email: "${data.email}",age: ${data.age},nom: "${data.lastname}",prenom: "${data.firstname}",adresse: "${data.adresse}",idVille: ${data.ville}, idParrain: ${data.parrainage}, photo: "${photo}"}) {
                  email,
                  age,
                  nom,
                  prenom,
                  adresse,
                  idVille,
                  idParrain,
                  photo
                }
              }
            `
          }),
        }).then(res => console.log(res.json()));
        console.log('submit!')
        new Notification(`Formulaire envoyé depuis la localisation suivante (${navigator.geolocation.getCurrentPosition(success, error, opt)}) !`)
      } else {
        console.log('error submit!', fields)
        new Notification('Erreur lors de la validation du formulaire !')
      }
    })
  }
}

const resetForm = (formEl) => {
  if (!formEl) return
  formEl.resetFields()
}

var opt = {
  enableHighAccuracy: true,
  timeout: 5000,
  maximumAge: 0
};

function success(pos) {
  var crd = pos.coords;

  new Notification(`Votre position actuelle est :`)
  new Notification(`Latitude : ${crd.latitude}`)
  new Notification(`Longitude : ${crd.longitude}`)
  new Notification(`La précision est de ${crd.accuracy} mètres.`)
}

function error(err) {
  new Notification(`ERREUR (${err.code}): ${err.message}`)
}
</script>

<style>
.logo {

  display: flex;
  align-items: center;
  justify-content: center;

  margin-top: 3rem;
  margin-bottom: 3rem;

}

img {

  width: 6rem;
  height: 6rem;

}
</style>