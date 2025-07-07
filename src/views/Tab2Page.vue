<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-title>Trajetos Salvos</ion-title>
      </ion-toolbar>
    </ion-header>
    <ion-content :fullscreen="true">
      <ion-header collapse="condense">
        <ion-toolbar>
          <ion-title size="large">Trajetos Salvos</ion-title>
        </ion-toolbar>
      </ion-header>

      <ion-list v-if="trajetos.length > 0">
        <ion-item-group>
          <ion-item-divider>
            <ion-label>{{ trajetos.length }} trajeto(s) salvo(s)</ion-label>
          </ion-item-divider>

          <ion-item
              v-for="(trajeto, index) in trajetos"
              :key="trajeto.id"
              button
              @click="visualizarTrajeto(trajeto)"
          >
            <ion-icon :icon="routeIcon" slot="start" color="primary"></ion-icon>
            <ion-label>
              <h2>Trajeto {{ index + 1 }}</h2>
              <p>{{ formatarData(trajeto.startTime) }}</p>
              <p v-if="trajeto.distanciaTotal">
                <ion-icon :icon="walkIcon" color="medium"></ion-icon>
                {{ formatarDistancia(trajeto.distanciaTotal) }}
              </p>
              <p v-if="trajeto.pontosTotal">
                <ion-icon :icon="locationIcon" color="medium"></ion-icon>
                {{ trajeto.pontosTotal }} ponto{{ trajeto.pontosTotal > 1 ? 's' : '' }}
                coletado{{ trajeto.pontosTotal > 1 ? 's' : '' }}
              </p>
            </ion-label>
            <ion-icon :icon="chevronForwardIcon" slot="end" color="medium"></ion-icon>
          </ion-item>
        </ion-item-group>
      </ion-list>

      <div v-else class="empty-state">
        <ion-icon :icon="mapIcon" color="medium" size="large"></ion-icon>
        <h2>Nenhum trajeto salvo</h2>
        <p>Vá para a aba "Mapa" e grave seu primeiro trajeto!</p>
        <ion-button fill="outline" @click="irParaMapa">
          <ion-icon :icon="addIcon" slot="start"></ion-icon>
          Gravar Trajeto
        </ion-button>
      </div>

      <ion-fab vertical="bottom" horizontal="end" slot="fixed" v-if="trajetos.length > 0">
        <ion-fab-button @click="confirmarLimpeza" color="danger" size="small">
          <ion-icon :icon="trashIcon"></ion-icon>
        </ion-fab-button>
      </ion-fab>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import {
  IonPage,
  IonHeader,
  IonToolbar,
  IonTitle,
  IonContent,
  IonList,
  IonItem,
  IonItemGroup,
  IonItemDivider,
  IonLabel,
  IonIcon,
  IonButton,
  IonFab,
  IonFabButton,
  alertController
} from '@ionic/vue';
import {chevronForward, map as mapOutline, walk, location, add, trash, navigate} from 'ionicons/icons';
import {onMounted, ref} from 'vue';
import {Storage} from '@ionic/storage';
import {useRouter} from 'vue-router';

const router = useRouter();
const store = new Storage();
store.create();

const trajetos = ref<any[]>([]);

const chevronForwardIcon = chevronForward;
const mapIcon = mapOutline;
const walkIcon = walk;
const locationIcon = location;
const addIcon = add;
const trashIcon = trash;
const routeIcon = navigate;

// pega trajetos salvos no storage
const carregarTrajetos = async () => {
  try {
    const trajetosExistentes = await store.get('trajetos') || '[]';
    trajetos.value = JSON.parse(trajetosExistentes);
  } catch (error) {
    console.error('erro ao carregar trajetos:', error);
    trajetos.value = [];
  }
};

const formatarData = (isoString: string): string => {
  const data = new Date(isoString);
  return data.toLocaleDateString('pt-BR') + ' às ' + data.toLocaleTimeString('pt-BR', {
    hour: '2-digit',
    minute: '2-digit'
  });
};

const formatarDistancia = (metros: number): string => {
  if (metros < 1000) {
    return `${metros.toFixed(0)}m`;
  }
  return `${(metros / 1000).toFixed(2)}km`;
};


// quando clica no trajeto, manda pro mapa
const visualizarTrajeto = (trajeto: any) => {
  localStorage.setItem('trajetoSelecionado', JSON.stringify(trajeto));
  router.push('/tabs/tab1');
};

const irParaMapa = () => {
  router.push('/tabs/tab1');
};

// pergunta se quer mesmo apagar tudo
const confirmarLimpeza = async () => {
  const alert = await alertController.create({
    header: 'Confirmar Limpeza',
    message: `Tem certeza que deseja apagar todos os ${trajetos.value.length} trajetos salvos? Esta ação não pode ser desfeita.`,
    buttons: [
      {
        text: 'Cancelar',
        role: 'cancel'
      },
      {
        text: 'Apagar Todos',
        role: 'destructive',
        handler: () => {
          limparTodosTrajetos();
        }
      }
    ]
  });

  await alert.present();
};

const limparTodosTrajetos = async () => {
  try {
    await store.remove('trajetos');
    trajetos.value = [];
  } catch (error) {
    console.error('erro ao limpar trajetos:', error);
  }
};

onMounted(() => {
  carregarTrajetos();
});

// recarrega quando volta pra aba
const recarregarQuandoVisivel = () => {
  if (document.visibilityState === 'visible') {
    carregarTrajetos();
  }
};

document.addEventListener('visibilitychange', recarregarQuandoVisivel);
</script>

<style scoped>
.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 60vh;
  text-align: center;
  padding: 2rem;
}

.empty-state ion-icon {
  margin-bottom: 1rem;
}

.empty-state h2 {
  color: var(--ion-color-medium);
  margin-bottom: 0.5rem;
}

.empty-state p {
  color: var(--ion-color-medium);
  margin-bottom: 2rem;
  max-width: 300px;
}
</style>
