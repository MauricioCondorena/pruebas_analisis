<template>
  <section class="contacto">
    <h2 class="titulo">Contacto / Acerca de</h2>

    <div class="card">
      <div class="foto-container">
        <img src="@/assets/Caballero.jpg" alt="Autor" class="foto" />
      </div>

      <p><b>Proyecto:</b> Análisis de Algoritmos</p>
      <p><b>Autor:</b> ALEJANDRO SAMIEL TRUJILLO CHUQUIMIA</p>
      <p><b>Universidad:</b> Universidad Católica Boliviana “San Pablo”</p>

      <hr class="linea" />

      <p><b>Correo:</b> alejandro.trujillo.c@ucb.edu.bo</p>
      <p><b>GitHub:</b> https://github.com/tuusuario</p>
      <p><b>Versión:</b> 1.0</p>
    </div>
  </section>
</template>

<script>
export default {
  name: "ContactoVista",
};
</script>

<style scoped>
.contacto {
  padding: 10px 0;
}

.titulo {
  margin: 0 0 12px;
  font-size: 24px;
  font-weight: 900;
  color: #0b1220;
}

.card {
  background: #fff;
  border: 1px solid rgba(15, 23, 42, 0.1);
  border-radius: 14px;
  padding: 16px;
  box-shadow: 0 10px 20px rgba(15, 23, 42, 0.06);
}

.linea {
  border: 0;
  height: 1px;
  background: rgba(15, 23, 42, 0.1);
  margin: 14px 0;
}

p {
  margin: 8px 0;
  color: #22324d;
}

.foto-container {
  display: flex;
  justify-content: center;
  margin-bottom: 18px;
}

.foto {
  width: 140px;
  height: 140px;
  object-fit: cover;
  border-radius: 50%;
  border: 4px solid #f5b400;
  box-shadow: 0 8px 20px rgba(15, 23, 42, 0.2);
}
</style>
