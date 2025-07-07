<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-title>Mapa</ion-title>
      </ion-toolbar>
    </ion-header>
    <ion-content :fullscreen="true">
      <div id="map-container">
        <div id="map" style="height: 100%; width: 100%;"></div>
      </div>

      <div v-if="trajetoSelecionado" class="trajeto-info">
        <span>📍 Mostrando trajeto selecionado da lista</span>
        <ion-button @click="fecharTrajetoSelecionado" fill="clear" size="small" class="close-button">
          <ion-icon :icon="closeIcon" slot="icon-only"></ion-icon>
        </ion-button>
      </div>

      <ion-fab v-if="!trajetoSelecionado" vertical="bottom" horizontal="end" slot="fixed">
        <ion-fab-button @click="toggleRecording" :color="isRecording ? 'danger' : 'primary'">
          <ion-icon :icon="isRecording ? stopIcon : playIcon"></ion-icon>
        </ion-fab-button>
      </ion-fab>

      <ion-fab v-if="!trajetoSelecionado" vertical="bottom" horizontal="start" slot="fixed">
        <ion-fab-button @click="irParaTrajetos" color="secondary">
          <ion-icon :icon="listIcon"></ion-icon>
        </ion-fab-button>
      </ion-fab>

      <ion-fab v-if="!trajetoSelecionado && !isRecording" vertical="bottom" horizontal="end" slot="fixed"
               style="margin-bottom: 65px;">
        <ion-fab-button @click="obterLocalizacaoAtual" color="light" size="small">
          <ion-icon :icon="locateIcon"></ion-icon>
        </ion-fab-button>
      </ion-fab>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import 'mapbox-gl/dist/mapbox-gl.css';
import {
  IonPage,
  IonHeader,
  IonToolbar,
  IonTitle,
  IonContent,
  IonFab,
  IonFabButton,
  IonIcon,
  IonButton
} from '@ionic/vue';
import {play, stop, list, close, locate} from 'ionicons/icons';
import mapboxgl from 'mapbox-gl';
import {onMounted, ref, onUnmounted} from "vue";
import {Storage} from "@ionic/storage";
import * as turf from '@turf/turf';
import {useRouter} from 'vue-router';

const router = useRouter();
const store = new Storage();
store.create();

let map: mapboxgl.Map;
const trajetoSelecionado = ref(false);
const isRecording = ref(false);
const trajetoAtual = ref<[number, number][]>([]);
let recordingInterval: NodeJS.Timeout | null = null;
let currentMarker: mapboxgl.Marker | null = null;
let userLocationMarker: mapboxgl.Marker | null = null;
let startTime: string | null = null;
let currentUserLocation: [number, number] | null = null;

const playIcon = play;
const stopIcon = stop;
const listIcon = list;
const closeIcon = close;
const locateIcon = locate;

// detecta quando volta pra aba
document.addEventListener('visibilitychange', () => {
  if (!document.hidden) {
    setTimeout(() => {
      verificarTrajetoSelecionado();
    }, 100);
  }
});

// detecta mudança de rota
router.afterEach((to) => {
  if (to.path === '/tabs/tab1') {
    setTimeout(() => {
      verificarTrajetoSelecionado();
    }, 200);
  }
});

const handleStorageChange = (event: StorageEvent) => {
  if (event.key === 'trajetoSelecionado' && event.newValue) {
    setTimeout(() => {
      verificarTrajetoSelecionado();
    }, 100);
  }
};

window.addEventListener('storage', handleStorageChange);

onMounted(async () => {
  verificarTrajetoSelecionado();

  mapboxgl.accessToken = 'pk.eyJ1IjoiZGFsc29jaGlvIiwiYSI6ImNtY21rZWt2dTBsa2gyam9yNnJkc29yZjMifQ.W67jYaMUXu2dIZqB2ifFCQ';

  map = new mapboxgl.Map({
    container: 'map',
    style: 'mapbox://styles/mapbox/streets-v12',
    center: [-74.5, 40],
    zoom: 9,
  });

  setTimeout(() => {
    map.resize();
    obterLocalizacaoAtual();
  }, 500);

  map.on('style.load', () => {
    map.addSource('trajeto-selecionado', {
      type: 'geojson',
      data: {
        type: 'Feature',
        properties: {},
        geometry: {
          type: 'LineString',
          coordinates: []
        }
      }
    });

    map.addLayer({
      id: 'trajeto-selecionado-line',
      type: 'line',
      source: 'trajeto-selecionado',
      layout: {
        'line-join': 'round',
        'line-cap': 'round'
      },
      paint: {
        'line-color': '#ff6b35',
        'line-width': 5,
        'line-opacity': 0.8
      }
    });

    verificarTrajetoSelecionado();
  });

  // controle nativo do mapbox para gps
  map.addControl(new mapboxgl.GeolocateControl({
    positionOptions: {
      enableHighAccuracy: true
    },
    trackUserLocation: true,
    showUserHeading: true,
    showUserLocation: true
  }));

  obterLocalizacaoAtual();
});

// verifica se veio algum trajeto da aba dos trajetos salvos
const verificarTrajetoSelecionado = () => {
  const trajetoSelecionadoStr = localStorage.getItem('trajetoSelecionado');

  if (trajetoSelecionadoStr) {
    try {
      const trajeto = JSON.parse(trajetoSelecionadoStr);
      trajetoSelecionado.value = true;
      mostrarTrajetoSelecionado(trajeto);
      localStorage.removeItem('trajetoSelecionado');
    } catch (error) {
      console.error('erro ao carregar trajeto:', error);
    }
  }
};

// exibe o trajeto selecionado no mapa
const mostrarTrajetoSelecionado = (trajeto: any) => {
  if (!map || !map.getSource('trajeto-selecionado')) {
    setTimeout(() => mostrarTrajetoSelecionado(trajeto), 500);
    return;
  }

  trajetoSelecionado.value = true;
  const coords = trajeto.coordinatesSimplified || trajeto.coordinates;

  if (coords && coords.length >= 1) {
    // se for só um ponto, só centraliza o mapa
    if (coords.length === 1) {
      const coord = coords[0];
      map.flyTo({
        center: coord as [number, number],
        zoom: 16,
        duration: 1000
      });

      new mapboxgl.Marker({color: '#3b82f6'})
          .setLngLat(coord as [number, number])
          .setPopup(new mapboxgl.Popup().setHTML('<strong>Localização</strong><br/>' + new Date(trajeto.startTime).toLocaleString('pt-BR')))
          .addTo(map);
      return;
    }

    try {
      // desenha a linha do trajeto
      const linhaTrajeto = turf.lineString(coords);
      const source = map.getSource('trajeto-selecionado') as mapboxgl.GeoJSONSource;
      source.setData(linhaTrajeto);

      const bbox = turf.bbox(linhaTrajeto);
      map.fitBounds(bbox as [number, number, number, number], {
        padding: 50,
        duration: 1000
      });

      const inicio = coords[0];
      const fim = coords[coords.length - 1];

      // marcador para início
      new mapboxgl.Marker({color: '#22c55e'})
          .setLngLat(inicio as [number, number])
          .setPopup(new mapboxgl.Popup().setHTML('<strong>Início</strong><br/>' + new Date(trajeto.startTime).toLocaleString('pt-BR')))
          .addTo(map);

      // marcador para fim
      new mapboxgl.Marker({color: '#ef4444'})
          .setLngLat(fim as [number, number])
          .setPopup(new mapboxgl.Popup().setHTML('<strong>Fim</strong><br/>' + new Date(trajeto.endTime).toLocaleString('pt-BR')))
          .addTo(map);
    } catch (error) {
      console.error('erro ao mostrar trajeto:', error);
    }
  }
};

onUnmounted(() => {
  window.removeEventListener('storage', handleStorageChange);
});

// fecha o trajeto que tava sendo exibido
const fecharTrajetoSelecionado = () => {
  trajetoSelecionado.value = false;

  const markers = document.querySelectorAll('.mapboxgl-marker');
  markers.forEach(marker => marker.remove());

  if (map && map.getSource('trajeto-selecionado')) {
    const source = map.getSource('trajeto-selecionado') as mapboxgl.GeoJSONSource;
    source.setData({
      type: 'Feature',
      properties: {},
      geometry: {
        type: 'LineString',
        coordinates: []
      }
    });
  }

  userLocationMarker = null;
  currentMarker = null;
  obterLocalizacaoAtual();
};

// botão de play/stop da gravação
const toggleRecording = async () => {
  if (isRecording.value) {
    isRecording.value = false;

    if (recordingInterval) {
      clearInterval(recordingInterval);
      recordingInterval = null;
    }

    if (trajetoAtual.value.length > 0) {
      await salvarTrajeto();
    }
  } else {
    isRecording.value = true;
    trajetoAtual.value = [];
    startTime = new Date().toISOString();

    // se já tem localização, usa ela na hora
    if (currentUserLocation) {
      trajetoAtual.value.push(currentUserLocation);
      currentMarker = new mapboxgl.Marker({color: '#ef4444'})
          .setLngLat(currentUserLocation)
          .addTo(map);
    }

    // remove o marcador azul
    if (userLocationMarker) {
      userLocationMarker.remove();
      userLocationMarker = null;
    }

    if (!currentUserLocation) {
      await salvarLocalizacaoAtual();
    }

    // salva localização a cada 3 segundos
    recordingInterval = setInterval(() => {
      salvarLocalizacaoAtual();
    }, 3000);
  }
};

// pega localização atual do gps
const salvarLocalizacaoAtual = async () => {
  try {
    const position = await new Promise<GeolocationPosition>((resolve, reject) => {
      navigator.geolocation.getCurrentPosition(resolve, reject, {
        enableHighAccuracy: true,
        timeout: 10000,
        maximumAge: 0
      });
    });

    const novaLocalizacao: [number, number] = [position.coords.longitude, position.coords.latitude];
    currentUserLocation = novaLocalizacao;

    // não salva se for muito perto da última
    if (trajetoAtual.value.length > 0) {
      const ultimaLocalizacao = trajetoAtual.value[trajetoAtual.value.length - 1];
      const distancia = turf.distance(ultimaLocalizacao, novaLocalizacao, {units: 'meters'});

      if (distancia < 10) {
        return;
      }
    }

    trajetoAtual.value.push(novaLocalizacao);

    if (currentMarker) {
      currentMarker.remove();
    }

    currentMarker = new mapboxgl.Marker({color: '#ef4444'})
        .setLngLat(novaLocalizacao)
        .addTo(map);

    map.flyTo({
      center: novaLocalizacao,
      zoom: 16,
      duration: 1000
    });
  } catch (error) {
    console.error('erro ao pegar localização:', error);
  }
};

// salva o trajeto quando para a gravação
const salvarTrajeto = async () => {
  if (trajetoAtual.value.length === 0) {
    return;
  }

  try {
    const endTime = new Date().toISOString();
    const pontosOriginais = trajetoAtual.value.length;

    let coordinatesSimplified = trajetoAtual.value;
    let distanciaTotal = 0;

    if (trajetoAtual.value.length > 2) {
      const linha = turf.lineString(trajetoAtual.value);
      const linhaSimplificada = turf.simplify(linha, {tolerance: 0.0001, highQuality: true});
      coordinatesSimplified = linhaSimplificada.geometry.coordinates as [number, number][];
      distanciaTotal = turf.length(linha, {units: 'meters'});
    }

    const novoTrajeto = {
      id: Date.now().toString(),
      startTime: startTime || endTime,
      endTime: endTime,
      coordinates: trajetoAtual.value,
      coordinatesSimplified,
      pontosTotal: pontosOriginais,
      pontosSimplificados: coordinatesSimplified.length,
      distanciaTotal
    };

    const trajetosExistentes = await store.get('trajetos') || '[]';
    const trajetos = JSON.parse(trajetosExistentes);
    trajetos.push(novoTrajeto);
    await store.set('trajetos', JSON.stringify(trajetos));

    trajetoAtual.value = [];
    startTime = null;

    if (currentMarker) {
      currentMarker.remove();
      currentMarker = null;
    }
  } catch (error) {
    console.error('erro ao salvar trajeto:', error);
  }
};

const irParaTrajetos = () => {
  router.push('/tabs/tab2');
};

// busca onde o usuário tá agora
const obterLocalizacaoAtual = () => {
  if (navigator.geolocation) {
    navigator.geolocation.getCurrentPosition(
        (position) => {
          const coords: [number, number] = [position.coords.longitude, position.coords.latitude];
          currentUserLocation = coords;

          map.flyTo({
            center: coords,
            zoom: 15,
            duration: 1000
          });

          if (userLocationMarker) {
            userLocationMarker.remove();
          }

          userLocationMarker = new mapboxgl.Marker({
            color: '#007AFF',
            scale: 0.8
          })
              .setLngLat(coords)
              .setPopup(new mapboxgl.Popup().setHTML('<strong>Sua localização atual</strong>'))
              .addTo(map);
        },
        (error) => {
          console.error('erro ao pegar localização:', error);
          // se não conseguir, vai pra são paulo
          map.flyTo({
            center: [-46.6333, -23.5505],
            zoom: 10,
            duration: 1000
          });
        },
        {
          enableHighAccuracy: true,
          timeout: 10000,
          maximumAge: 0
        }
    );
  }
};
</script>

<style scoped>
#map {
  position: absolute;
  top: 0;
  bottom: 0;
  width: 100%;
  height: 100%;
}

.trajeto-info {
  position: absolute;
  top: 65px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 8px 16px;
  border-radius: 16px;
  font-size: 14px;
  z-index: 2000;
  display: flex;
  align-items: center;
  gap: 8px;
}

.close-button {
  --color: white;
  --padding-start: 4px;
  --padding-end: 4px;
  margin: 0;
  height: 24px;
  width: 24px;
}
</style>
