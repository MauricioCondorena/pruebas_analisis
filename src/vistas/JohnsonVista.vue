<template>
  <section class="matriz">
    <h2 class="titulo">JHONSON</h2>

    <div class="acciones">
      <button class="btn-sec" @click="$emit('volver')">
        Volver al grafo
      </button>

      <button class="btn" @click="resolverJohnson">
        Calcular camino crítico
      </button>

      <button class="btn-sec" @click="limpiarGrafo">
        Limpiar
      </button>

      <button class="btn-sec" @click="exportarJSON">
        Exportar JSON
      </button>

      <button class="btn-sec" @click="triggerImport">
        Importar JSON
      </button>

      <button class="btn btn-sec" @click="abrirAyuda">
        Ayuda
      </button>

      <input ref="fileInput" type="file" accept="application/json,.json" style="display: none" @change="importarJSON" />

      <select v-model="modo" class="select-modo">
        <option value="min">Minimizar</option>
        <option value="max">Maximizar</option>
      </select>

    </div>

    <!-- 🔥 GRAFO -->
    <div class="lienzo" ref="networkEl"></div>

    <!-- RESULTADO -->
    <div v-if="resultado.length" class="resultado-card">
      <h3>Camino crítico</h3>
      <div class="camino">
        <span v-for="(n, i) in resultado" :key="i">
          <b>{{ n }}</b>
          <span v-if="i < resultado.length - 1"> → </span>
        </span>
      </div>

      <h3>Holguras</h3>
      <ul>
        <li v-for="h in holguras" :key="h.id">
          <b>{{ h.label }}</b> → Holgura: {{ h.holgura }}
        </li>
      </ul>
    </div>
    <div v-if="modal.abierto" class="modal-backdrop" @click.self="cerrarModal">
      <div class="modal">
        <div class="modal-head">
          <h3 class="modal-title">{{ modal.titulo }}</h3>
          <button class="modal-x" @click="cerrarModal">×</button>
        </div>

        <div class="modal-body">

          <!-- 🔥 AYUDA -->
          <div v-if="modal.tipo === 'ayuda'" class="ayuda-modal">
            <p><b>Cómo usar la vista</b></p>

            <ul class="help-list">
              <li>Doble click para crear un nodo.</li>
              <li>Elige <b>ORIGEN</b> o <b>DESTINO</b>.</li>
              <li>Conecta: <b>ORIGEN → DESTINO</b> y asigna un peso.</li>
            </ul>

            <p><b>Qué hace el sistema</b></p>

            <ul class="help-list">
              <li>Genera nodos y su respectiva solución automáticamente.</li>
              <li><b>Minimizar</b> → menor valor.</li>
              <li><b>Maximizar</b> → mayor valor.</li>
            </ul>

            <p><b>Extras</b></p>

            <ul class="help-list">
              <li>Exportar / Importar el grafo en JSON.</li>
              <li>Se agregan nodos <b>dummy</b> si falta balance.</li>
            </ul>
          </div>

          <!-- 🔹 FORMULARIO NORMAL -->
          <div v-else>
            <label class="modal-label">{{ modal.label }}</label>

            <input class="modal-input" v-model="modal.valor" :placeholder="modal.placeholder"
              @keydown.enter.prevent="confirmarModal" />

            <p class="modal-hint">{{ modal.hint }}</p>
            <p v-if="modal.error" class="modal-error">{{ modal.error }}</p>
          </div>
        </div>

        <div class="modal-actions">
          <button class="btn-sec" @click="cerrarModal">
            {{ modal.tipo === 'ayuda' ? 'Cerrar' : 'Cancelar' }}
          </button>

          <button v-if="modal.tipo !== 'ayuda'" class="btn" @click="confirmarModal">
            Guardar
          </button>
        </div>
      </div>
    </div>
    <div v-else-if="modal.tipo === 'ayuda'" class="modal-body">
      <div class="ayuda-modal">
        <p><b>Cómo usar la vista</b></p>

        <ul class="help-list">
          <li>Doble click para crear un nodo.</li>
          <li>Elige <b>ORIGEN</b> o <b>DESTINO</b>.</li>
          <li>Conecta: <b>ORIGEN → DESTINO</b> y asigna un peso.</li>
        </ul>

        <p><b>Qué hace el sistema</b></p>

        <ul class="help-list">
          <li>Genera matrices y solución automáticamente.</li>
          <li><b>Minimizar</b> → menor valor.</li>
          <li><b>Maximizar</b> → mayor valor.</li>
        </ul>

        <p><b>Extras</b></p>

        <ul class="help-list">
          <li>Exportar / Importar el grafo en JSON.</li>
          <li>Se agregan nodos <b>dummy</b> si falta balance.</li>
        </ul>
      </div>
    </div>
  </section>
</template>

<script>
import { Network } from "vis-network/standalone";
import { DataSet } from "vis-data/standalone";

export default {
  name: "JohnsonVista",

  data() {
    return {
      resultado: [],
      holguras: [],
      network: null,
      nodesData: null,
      edgesData: null,
      modo: "max", // "max" o "min"

      contadorNodos: 0,
      contadorAristas: 0,

      nombreNodo: "",
      pesoArista: 1,
      origenId: null,
      pendienteArista: null,

      modal: {
        abierto: false,
        tipo: "",
        titulo: "",
        label: "",
        placeholder: "",
        hint: "",
        valor: "",
        error: "",
        payload: null,
      },
    };
  },

  mounted() {
    this.nodesData = new DataSet([]);
    this.edgesData = new DataSet([]);

    this.network = new Network(
      this.$refs.networkEl,
      { nodes: this.nodesData, edges: this.edgesData },
      {
        physics: false,

        interaction: {
          dragNodes: true,
          dragView: false,
          zoomView: false,
        },

        nodes: {
          shape: "box",
          margin: 16,
          borderWidth: 1.5,
          color: {
            background: "#1e293b",
            border: "#475569",
          },
          font: {
            color: "#e2e8f0",
            size: 16,
            align: "center",
            multi: true
          },
          shadow: {
            enabled: true,
            color: "rgba(0,0,0,0.6)",
            size: 15,
            x: 0,
            y: 6
          }
        },

        edges: {
          arrows: {
            to: {
              enabled: true,
              scaleFactor: 0.7,
            },
          },
          width: 2,
          color: {
            color: "#334155",
            highlight: "#f97316",
            hover: "#f97316",
            inherit: false,
          },
          smooth: {
            enabled: true,
            type: "cubicBezier",
            roundness: 0.45,
          },
          shadow: {
            enabled: true,
            color: "rgba(0,0,0,0.5)",
            size: 6
          }
        },
      }
    );

    this.network.on("doubleClick", (params) => {
      if (!params.pointer) return;

      this.abrirModalCrearNodo(
        params.pointer.canvas.x,
        params.pointer.canvas.y
      );
    });

    this.network.on("click", (params) => {
      if (params.nodes.length) {
        const nodeId = params.nodes[0];
        this.clickNodo(nodeId);
      }
    });
  },
  methods: {
    abrirModalCrearNodo(x, y) {
      this.modal = {
        abierto: true,
        tipo: "crearNodo",
        titulo: "Crear nodo",
        label: "Nombre del nodo",
        placeholder: "Ej: A, B, Casa, Nodo1",
        hint: "Reglas: 1–12 caracteres, sin duplicados.",
        valor: "",
        error: "",
        payload: { x, y },
      };
    },

    triggerImport() {
      const el = this.$refs.fileInput;
      if (!el) {
        alert("No se encontró el input file.");
        return;
      }
      el.value = "";
      el.click();
    },

    exportarJSON() {
      try {
        let nombre = prompt("Nombre del archivo:", "johnson_grafo");
        if (nombre === null) return;

        nombre = nombre.trim() || "johnson_grafo";

        const payload = {
          version: 1,
          exportedAt: new Date().toISOString(),
          state: {
            contadorNodos: this.contadorNodos,
            contadorAristas: this.contadorAristas,
            nodes: this.nodesData.get(),
            edges: this.edgesData.get()
          }
        };

        const jsonStr = JSON.stringify(payload, null, 2);
        const blob = new Blob([jsonStr], { type: "application/json" });
        const url = URL.createObjectURL(blob);

        const a = document.createElement("a");
        a.href = url;
        a.download = `${nombre}.json`;
        a.click();

        URL.revokeObjectURL(url);
      } catch (e) {
        console.error(e);
        alert("Error al exportar JSON");
      }
    },

    async importarJSON(evt) {
      try {
        const file = evt?.target?.files?.[0];
        if (!file) return;

        const text = await file.text();
        const data = JSON.parse(text);

        const state = data?.state;
        if (!state || !Array.isArray(state.nodes) || !Array.isArray(state.edges)) {
          alert("JSON inválido");
          return;
        }

        // 🔥 LIMPIAR TODO
        this.nodesData.clear();
        this.edgesData.clear();
        this.resultado = [];
        this.holguras = [];

        // 🔹 NODOS
        this.nodesData.add(
          state.nodes.map(n => ({
            ...n,
            id: Number(n.id)
          }))
        );

        // 🔹 ARISTAS
        this.edgesData.add(
          state.edges.map(e => ({
            ...e,
            id: Number(e.id),
            from: Number(e.from),
            to: Number(e.to),
            arrows: {
              to: { enabled: true, scaleFactor: 0.7 }
            }
          }))
        );

        // 🔹 CONTADORES
        const maxNode = state.nodes.length
          ? Math.max(...state.nodes.map(n => Number(n.id)))
          : 0;

        const maxEdge = state.edges.length
          ? Math.max(...state.edges.map(e => Number(e.id)))
          : 0;

        this.contadorNodos = Math.max(state.contadorNodos || 0, maxNode);
        this.contadorAristas = Math.max(state.contadorAristas || 0, maxEdge);

        // 🔥 REDIBUJAR
        this.network.fit({ animation: { duration: 300 } });

      } catch (e) {
        console.error(e);
        alert("Error al importar JSON");
      } finally {
        if (evt?.target) evt.target.value = "";
      }
    },

    abrirModalCrearAristaPeso(from, to) {
      this.modal = {
        abierto: true,
        tipo: "crearArista",
        titulo: "Crear arista",
        label: `Peso (${from} → ${to})`,
        placeholder: "Ej: 1, 2.5, -3",
        hint: "Acepta entero o decimal.",
        valor: "1",
        error: "",
        payload: { from, to },
      };
    },

    abrirAyuda() {
      this.modal = {
        abierto: true,
        tipo: "ayuda",
        titulo: "Ayuda Jhonson",
        nombre: "",
        tipoNodo: "origen",
        peso: "",
        x: 0,
        y: 0,
        from: null,
        to: null,
        desdeLabel: "",
        hastaLabel: "",
      };
    },

    cerrarModal() {
      this.modal.abierto = false;
      this.modal.error = "";
    },

    clickNodo(destinoId) {
      if (this.origenId === null) {
        this.origenId = destinoId;
        return;
      }

      const from = this.origenId;
      const to = destinoId;

      this.origenId = null;

      const pregunta = `¿La arista será dirigida? (SI/NO)`;
      let resp = window.prompt(pregunta, "SI");

      if (resp == null) return;

      resp = String(resp).trim().toUpperCase();

      const esSi = resp === "SI" || resp === "S";
      const esNo = resp === "NO" || resp === "N";

      if (!esSi && !esNo) {
        alert("Respuesta inválida");
        return;
      }

      this.pendienteArista = { from, to, directed: esSi };
      this.abrirModalCrearAristaPeso(from, to);
    },

    confirmarModal() {
      // 🟢 CREAR NODO
      if (this.modal.tipo === "crearNodo") {
        const nombre = this.modal.valor.trim();
        if (!nombre) return;

        this.contadorNodos++;

        this.nodesData.add({
          id: this.contadorNodos,
          label: nombre,
          x: this.modal.payload.x,
          y: this.modal.payload.y,
        });

        this.cerrarModal();
      }

      // 🔵 CREAR ARISTA (🔥 SIN arrows aquí)
      if (this.modal.tipo === "crearArista") {
        const peso = this.modal.valor;
        if (!peso) return;

        this.contadorAristas++;

        this.edgesData.add({
          id: this.contadorAristas,
          from: this.modal.payload.from,
          to: this.modal.payload.to,
          label: peso,
          arrows: {
            to: {
              enabled: true,
              scaleFactor: 0.7
            }
          }
        });

        this.cerrarModal();
      }
    },

    resolverJohnson() {
      const nodos = this.nodesData.get();
      const aristas = this.edgesData.get();

      if (!nodos.length) return;

      // 🔹 ADYACENCIA
      const adj = {};
      nodos.forEach(n => adj[n.id] = []);

      aristas.forEach(e => {
        adj[e.from].push({
          to: e.to,
          peso: Number(e.label || 0)
        });
      });

      // 🔹 IN-DEGREE
      const inDegree = {};
      nodos.forEach(n => inDegree[n.id] = 0);
      aristas.forEach(e => inDegree[e.to]++);

      const inDegreeOriginal = { ...inDegree };

      // 🔹 TOPOLOGÍA
      const cola = [];
      nodos.forEach(n => {
        if (inDegree[n.id] === 0) cola.push(n.id);
      });

      const orden = [];

      while (cola.length) {
        const actual = cola.shift();
        orden.push(actual);

        adj[actual].forEach(ar => {
          inDegree[ar.to]--;
          if (inDegree[ar.to] === 0) cola.push(ar.to);
        });
      }

      // 🔹 DISTANCIAS (ET)
      const dist = {};
      const prev = {};

      nodos.forEach(n => {
        dist[n.id] = this.modo === "max" ? -Infinity : Infinity;
        prev[n.id] = null;
      });

      // 🔹 NODOS INICIO
      orden.forEach(id => {
        if (inDegreeOriginal[id] === 0) {
          dist[id] = 0;
        } else {
          dist[id] = this.modo === "max" ? -Infinity : Infinity;
        }
      });

      // 🔥 FORWARD
      orden.forEach(id => {
        adj[id].forEach(ar => {

          if (dist[id] === Infinity || dist[id] === -Infinity) return;

          const nuevo = dist[id] + ar.peso;

          if (this.modo === "max") {
            if (nuevo > dist[ar.to]) {
              dist[ar.to] = nuevo;
              prev[ar.to] = id;
            }
          } else {
            if (nuevo < dist[ar.to]) {
              dist[ar.to] = nuevo;
              prev[ar.to] = id;
            }
          }
        });
      });

      // 🔥 🔥 🔥 CORRECCIÓN CLAVE AQUÍ 🔥 🔥 🔥
      // 🔹 EXTREMO = NODO FINAL (SIN SALIDAS)
      let extremo = null;

      nodos.forEach(n => {
        if (adj[n.id].length === 0) { // nodo final
          if (
            extremo === null ||
            (
              this.modo === "max"
                ? dist[n.id] > dist[extremo]
                : dist[n.id] < dist[extremo]
            )
          ) {
            extremo = n.id;
          }
        }
      });

      if (extremo === null) return;

      // 🔹 CAMINO
      const camino = [];
      let actual = extremo;

      while (actual !== null) {
        const nodo = nodos.find(n => n.id === actual);
        camino.unshift(nodo.label.split("\n")[0]);
        actual = prev[actual];
      }

      this.resultado = camino;


      const LT_real = {};

      // inicializar
      nodos.forEach(n => {
        if (n.id === extremo) {
          LT_real[n.id] = dist[extremo];
        } else {
          LT_real[n.id] = Infinity;
        }
      }); 

      // backward COMPLETO (todos los caminos)
      [...orden].reverse().forEach(id => {
        adj[id].forEach(ar => {
          LT_real[id] = LT_real[ar.to] - ar.peso;
        });
      });

      // 🔥 BACKWARD (LT)
      const LT = {};

      nodos.forEach(n => {
        if (n.id === extremo) {
          LT[n.id] = dist[extremo];
        } else {
          LT[n.id] = Infinity;
        }
      });

      [...orden].reverse().forEach(id => {
        adj[id].forEach(ar => {

          const nodoFrom = nodos.find(n => n.id === id);
          const nodoTo = nodos.find(n => n.id === ar.to);

          const fromLabel = nodoFrom.label.split("\n")[0];
          const toLabel = nodoTo.label.split("\n")[0];

          const enCamino =
            this.resultado.includes(fromLabel) &&
            this.resultado.includes(toLabel) &&
            this.resultado.indexOf(toLabel) === this.resultado.indexOf(fromLabel) + 1;

          if (enCamino) {
            LT[id] = LT[ar.to] - ar.peso;
          }

        });
      });

      // 🔥 HOLGURAS (FORMULA INGENIERO)
      this.holguras = [];

      aristas.forEach(e => {
        const peso = Number(e.label || 0);

        const h = LT[e.to] - dist[e.from] - peso;

        this.holguras.push({
          from: e.from,
          to: e.to,
          holgura: h
        });
      });

      // 🔹 ACTUALIZAR NODOS
      nodos.forEach(n => {

        const nombre = n.label.split("\n")[0];
        const esCritico = this.resultado.includes(nombre);

        let ltMostrar;
        let hNodo;

        if (esCritico) {
          // 🔥 camino crítico
          ltMostrar = LT[n.id];
          hNodo = LT[n.id] - dist[n.id];
        } else {
          // 🔥 fuera del camino crítico
          ltMostrar = LT_real[n.id];
          hNodo = LT_real[n.id] - dist[n.id];
        }

        this.nodesData.update({
          id: n.id,
          label: `${nombre}\n${dist[n.id]} | ${ltMostrar}\nH: ${hNodo}`,
          font: { align: "center" }
        });

      });

      this.animarCamino();
    },

    limpiarGrafo() {
      this.nodesData.clear();
      this.edgesData.clear();
      this.resultado = [];
      this.holguras = [];
      this.contadorNodos = 0;
      this.contadorAristas = 0;
    },

    async animarCamino() {
      const camino = this.resultado;

      const COLOR_BASE = "#334155";
      const COLOR_ACTIVO = "#f97316";
      const COLOR_GLOW = "#fb923c";

      const edgesActivas = new Set();

      // RESET
      this.nodesData.forEach(n => {
        this.nodesData.update({
          id: n.id,
          color: {
            background: "#1e293b",
            border: "#475569"
          },
          shadow: false
        });
      });

      this.edgesData.forEach(e => {
        this.edgesData.update({
          id: e.id,
          color: {
            color: COLOR_BASE,
            inherit: false
          },
          width: 2,
          dashes: false,
          shadow: false
        });
      });

      this.network.redraw();

      // 🔥 ANIMACIÓN SUPER GOD
      for (let i = 0; i < camino.length; i++) {
        const actual = camino[i];

        // 🟠 NODO ACTIVO (NEÓN)
        const nodo = this.nodesData.get().find(n =>
          n.label.startsWith(actual)
        );

        if (nodo) {
          this.nodesData.update({
            id: nodo.id,
            color: {
              background: "#7c2d12", // oscuro naranja
              border: COLOR_ACTIVO
            },
            shadow: {
              enabled: true,
              color: COLOR_GLOW,
              size: 40
            }
          });
        }

        // ⚡ ARISTA CON PARTÍCULAS NEÓN
        if (i < camino.length - 1) {
          const siguiente = camino[i + 1];

          const arista = this.edgesData.get().find(e =>
            this.nodesData.get(e.from).label.startsWith(actual) &&
            this.nodesData.get(e.to).label.startsWith(siguiente)
          );

          if (arista) {
            edgesActivas.add(arista.id);

            let frame = 0;

            const anim = setInterval(() => {
              frame++;

              edgesActivas.forEach(id => {
                this.edgesData.update({
                  id,
                  color: {
                    color: COLOR_ACTIVO,
                    inherit: false
                  },
                  width: 5 + Math.sin(frame * 0.4) * 2, // 🔥 pulso
                  dashes: [4, 12], // 🔥 partículas más visibles
                  shadow: {
                    enabled: true,
                    color: COLOR_GLOW,
                    size: 20 + Math.sin(frame * 0.4) * 5
                  }
                });
              });

              this.network.redraw();

              if (frame > 30) clearInterval(anim);
            }, 40);
          }
        }

        await new Promise(res => setTimeout(res, 700));
      }

      // 🔥 FINAL LIMPIO
      edgesActivas.forEach(id => {
        this.edgesData.update({
          id,
          dashes: false,
          width: 6,
          shadow: {
            enabled: true,
            color: COLOR_GLOW,
            size: 15
          }
        });
      });

      this.network.redraw();
    }
  }
};
</script>

<style src="@/estilos/johnsonVista.css"></style>
<style scoped src="@/estilos/grafoVista.css"></style>
