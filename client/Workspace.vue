<script setup>
import { ref, onMounted } from "vue";
import { api, act, base64 } from "./api";
const useLlm = ref(false);
const documents = ref([]),
  name = ref("Proje notları"),
  text = ref(
    "Bu proje Vue arayüzü ve Node.js sunucusundan oluşur. Veriler SQLite veritabanında saklanır. API istekleri oturum kontrolünden geçer. PostgreSQL geçişi sonraki geliştirme adımıdır.",
  ),
  file = ref(null),
  selected = ref(""),
  question = ref("Veriler nerede saklanır?"),
  answer = ref(null),
  busy = ref(false);
async function load() {
  documents.value = await api("/documents");
  if (!selected.value) selected.value = documents.value[0]?.id || "";
}
async function upload() {
  busy.value = true;
  try {
    const data = file.value
      ? { name: file.value.name, base64: await base64(file.value) }
      : { name: name.value, text: text.value };
    await api("/documents", "POST", data);
    await load();
  } finally {
    busy.value = false;
  }
}
onMounted(() => act(load, ""));
</script>
<template>
  <div class="alert">
    Varsayılan yerel kaynak arama modu. İsteğe bağlı Gemini yanıtı için sunucuda
    anahtar ve model tanımlanmalıdır. PDF sınırı: 5 MB, ilk 30 sayfa, 100.000
    karakter.
  </div>
  <div class="grid">
    <form class="panel" @submit.prevent="act(upload, 'Belge işlendi')">
      <h2>Belge ekle</h2>
      <label
        >Metin tabanlı PDF · en fazla 5 MB<input
          type="file"
          accept="application/pdf"
          @change="file = $event.target.files[0] || null"
      /></label>
      <p class="muted">PDF seçmediysen aşağıdaki metin kullanılır.</p>
      <label>Belge adı<input v-model="name" /></label
      ><label>Metin<textarea v-model="text" rows="9"></textarea></label
      ><button :disabled="busy">
        {{ busy ? "İşleniyor..." : "Parçala ve kaydet" }}
      </button>
    </form>
    <section class="panel">
      <form
        @submit.prevent="
          act(async () => {
            answer = await api('/documents/' + selected + '/ask', 'POST', {
              question,
              useLlm,
            });
          }, 'Kaynaklar getirildi')
        "
      >
        <h2>Belgede soru ara</h2>
        <label
          >Belge<select v-model="selected">
            <option v-for="d in documents" :value="d.id">
              {{ d.name }} · {{ d.chunks }} parça
            </option>
          </select></label
        ><label>Soru<input v-model="question" required /></label
        ><label class="check"
          ><input type="checkbox" v-model="useLlm" />Eşleşen parçaları Gemini
          servisine gönder ve yanıt üret (API kullanımı ücretli olabilir)</label
        ><button :disabled="!selected">Kaynakları bul</button>
      </form>
      <template v-if="answer"
        ><p>{{ answer.answer }}</p>
        <article class="card" v-for="s in answer.sources">
          <span class="badge"
            >Kaynak parça {{ s.chunk }} · eşleşme {{ s.score }}</span
          >
          <p>{{ s.text }}</p>
        </article></template
      >
    </section>
  </div>
  <section class="panel">
    <h2>Belgelerim</h2>
    <div v-for="d in documents" class="card">
      {{ d.name }}
      <button
        class="ghost small"
        @click="
          act(async () => {
            await api('/documents/' + d.id, 'DELETE');
            selected = '';
            answer = null;
            await load();
          })
        "
      >
        Sil
      </button>
    </div>
  </section>
</template>
