<template>
  <!-- Importamos una fuente más llamativa desde Google Fonts -->
  <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@400;700;900&display=swap" rel="stylesheet">
  
  <div class="app">
    <header class="header">
      <h1 class="logo-text">RAPIDO <span>&</span> RICO</h1>
      <div class="header-right">
        <div class="mesa-info"> 30 Productos</div>
        <button class="cart-btn-toggle" v-on:click="mostrarCarrito = true">
          🛒 <span class="btn-text">PEDIDO</span> 
          <span class="cart-badge">{{ pedidoItems.length }}</span>
        </button>
      </div>
    </header>

    <main class="dashboard">
      <section class="card">
        <h2 class="section-title">¡ELIGE TU FAVORITO!</h2>
        
        <div class="categorias">
          <button
            v-for="cat in categorias"
            :key="cat"
            class="cat-btn"
            :class="{ active: categoriaActiva === cat }"
            v-on:click="categoriaActiva = cat"
          >
            {{ cat.toUpperCase() }}
          </button>
        </div>

        <div class="productos-grid">
          <div
            v-for="p in obtenerProductosFiltrados()"
            :key="p.id"
            class="producto-card"
            v-on:click="agregarAlPedido(p)"
          >
            <div class="producto-imagen" :style="{ backgroundImage: 'url(' + p.imagen + ')' }"></div>
            
            <div class="producto-info">
              <strong class="producto-nombre">{{ p.nombre }}</strong>
              <small class="descripcion">{{ p.descripcion }}</small>
              <div class="precio-inline">${{ p.precio }}</div>
              <div class="add-btn-style">Agregar Al Carrito</div>
            </div>
          </div>
        </div>
      </section>

      <div class="modal-overlay" v-if="mostrarCarrito" v-on:click.self="mostrarCarrito = false">
        <section class="card modal-content">
          <div class="order-header">
            <h2 class="modal-title">MI PEDIDO 🍔</h2>
            <button class="close-modal" v-on:click="mostrarCarrito = false">✕</button>
          </div>
          
          <div class="mesa-selector">
            <label>NÚMERO DE MESA:</label>
            <input type="number" v-model="numeroMesa" min="1" max="99" />
          </div>

          <div class="lista-pedido" v-if="pedidoItems.length > 0">
            <div v-for="(item, idx) in pedidoItems" :key="idx" class="item-pedido">
              <div class="item-miniatura" :style="{ backgroundImage: 'url(' + item.imagen + ')' }"></div>
              <div class="item-info">
                <span class="item-nombre">{{ item.nombre }}</span>
                <span class="item-subtotal">${{ item.precio * item.cantidad }}</span>
              </div>
              <div class="item-controles">
                <div class="cantidad-box">
                  <button class="qty-btn" v-on:click="cambiarCantidad(item, -1)">-</button>
                  <span class="qty-num">{{ item.cantidad }}</span>
                  <button class="qty-btn" v-on:click="cambiarCantidad(item, 1)">+</button>
                </div>
                <button class="del-btn" v-on:click="eliminarItem(item)">🗑️</button>
              </div>
            </div>
          </div>

          <div v-else class="pedido-vacio">
            <p>EL CARRITO ESTÁ VACÍO</p>
            <span>¡Agrega algo rico ahora!</span>
          </div>

          <div class="order-total">
            <span>TOTAL:</span>
            <span class="total-monto">${{ obtenerTotal() }}</span>
          </div>

          <div class="btn-group">
            <button class="btn btn-enviar" v-on:click="enviarPedido">CONFIRMAR PEDIDO</button>
            <button class="btn btn-limpiar" v-on:click="limpiarPedido">VACIAR</button>
          </div>
        </section>
      </div>
    </main>

    <div v-show="mostrarAlerta" class="alert-box">
      {{ mensajeAlerta }}
    </div>

    <div style="display: none;">
      <div id="factura-pdf" class="factura-container">
        <div class="factura-header">
          <h2>RÁPIDO & RICO</h2>
          <p>Comprobante de Pedido Digital</p>
        </div>
        
        <div class="factura-meta">
          <p><strong>Fecha:</strong> {{ nuevaFecha }}</p>
          <p><strong>Mesa:</strong> {{ numeroMesa }}</p>
          <p><strong>Estado:</strong> Confirmado 🚀</p>
        </div>

        <table class="factura-tabla">
          <thead>
            <tr>
              <th>Cant.</th>
              <th>Producto</th>
              <th>Precio Un.</th>
              <th>Subtotal</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in itemsParaFactura" :key="item.id">
              <td>{{ item.cantidad }}x</td>
              <td>{{ item.nombre }}</td>
              <td>${{ item.precio }}</td>
              <td>${{ item.precio * item.cantidad }}</td>
            </tr>
          </tbody>
        </table>

        <div class="factura-total">
          <span>TOTAL A PAGAR:</span>
          <strong>${{ totalFactura }}</strong>
        </div>

        <div class="factura-footer">
          <p>¡Gracias por tu compra!</p>
          <p>Tu pedido ya se está preparando en la cocina.</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue";

const categoriaActiva = ref("Todos");
const numeroMesa = ref(1);
const mostrarAlerta = ref(false);
const mensajeAlerta = ref("");
const pedidoItems = ref([]);
const mostrarCarrito = ref(false);

const itemsParaFactura = ref([]);
const totalFactura = ref(0);
const nuevaFecha = ref("");

const categorias = ref(["Todos", "Hamburguesas", "Perros", "Salchipapas", "Acompañamientos", "Bebidas", "Especialidades"]);

const productos = ref([
  { id: 1, nombre: " Hamburguesa Simple", precio: 10500, descripcion: "Carne 150g, lechuga, tomate", categoria: "Hamburguesas", imagen: "https://img.hogar.mapfre.es/wp-content/uploads/2018/09/hamburguesa-sencilla.jpg" },
  { id: 2, nombre: " Hamburguesa Doble", precio: 13000, descripcion: "Doble carne, queso cheddar", categoria: "Hamburguesas", imagen: "https://thumbs.dreamstime.com/b/burger-doble-con-tocino-en-placa-de-pizarra-hamburguesa-dos-patatas-ternera-queso-cheddar-crujiente-y-lechuga-verde-fresca-388435375.jpg" },
  { id: 3, nombre: " Hamburguesa BBQ", precio: 16500, descripcion: "Salsa BBQ, cebolla crispy", categoria: "Hamburguesas", imagen: "https://www.shutterstock.com/image-photo/beef-burger-bbq-sauce-bacon-600nw-2471612047.jpg" },
  { id: 4, nombre: " Hamburguesa Hawaiana", precio: 15000, descripcion: "Piña, jamón, queso", categoria: "Hamburguesas", imagen: "https://thumbs.dreamstime.com/b/hamburguesa-hawaiana-con-la-pi%C3%B1a-el-jam%C3%B3n-y-queso-107562948.jpg" },
  { id: 5, nombre: " Hamburguesa Mexicana", precio: 18500, descripcion: "Guacamole, jalapeños", categoria: "Hamburguesas", imagen: "https://www.vvsupremo.com/wp-content/uploads/2018/05/Mexican-Burger-with-Chorizo.jpg" },
  { id: 6, nombre: " Perro Caliente", precio: 5000, descripcion: "Salchicha, papitas, salsas", categoria: "Perros", imagen: "https://media.istockphoto.com/id/1161043951/es/foto/perro-colombiano-paisa.jpg?s=612x612&w=0&k=20&c=-5Xi0DKEPupdcMSDJoymfhCIDVEgAjbHxohGu3ldwfE=" },
  { id: 7, nombre: " Perro Especial", precio: 8500, descripcion: "Tocineta, queso, piña", categoria: "Perros", imagen: "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcRNiaRNJ1ZXzBSvcjB9jpBhpzqJEkK8-6htGg&s" },
  { id: 8, nombre: " Perro Americano", precio: 10500, descripcion: "Chili con carne, queso", categoria: "Perros", imagen: "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTF8UbHxjp_hq5NviYymYMk1Tk4Ar9EFX6__A&s" },
  { id: 9, nombre: " Perro Ranchero", precio: 12000, descripcion: "Maíz, mayonesa chipotle", categoria: "Perros", imagen: "https://0201.nccdn.net/1_2/000/000/192/ea4/perro-ranchero.jpg" },
  { id: 10, nombre: " Perro Colombiano", precio: 10500, descripcion: "Piña, papa criolla", categoria: "Perros", imagen: "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTLxDX474LGREwRLHZfkQAEYVZVWQSBewxU6w&s" },
  { id: 11, nombre: " Salchipapa Clásica", precio: 9500, descripcion: "Papas fritas, salchicha, salsa rosada", categoria: "Salchipapas", imagen: "https://imag.bonviveur.com/emplatado-final-de-las-salchipapas.jpg" },
  { id: 12, nombre: " Salchipapa Queso", precio: 9500, descripcion: "Papas, salchicha, queso mozzarella gratinado", categoria: "Salchipapas", imagen: "https://lariendagastrofonda.com/wp-content/uploads/2023/08/salchipapas.png" },
  { id: 13, nombre: " Salchipapa Especial", precio: 14500, descripcion: "Papas, salchicha, tocineta, chorizo", categoria: "Salchipapas", imagen: "https://cloudfront-us-east-1.images.arcpublishing.com/infobae/CRU7IWBQBVBWTDZT2AZFIXREVI.jpg" },
  { id: 14, nombre: " Salchipapa Hawaiana", precio: 12000, descripcion: "Papas, salchicha, piña, queso", categoria: "Salchipapas", imagen: "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcSWV0MgnQr5PxvSyj1xBeQiNI27p1bKJ-BASQ&s" },
  { id: 15, nombre: " Salchipapa BBQ", precio: 16000, descripcion: "Papas, salchicha, salsa BBQ, pollo desmechado", categoria: "Salchipapas", imagen: "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcSrAF0C-i3JBXDkLAzYBHJODFdEszWg&s" },
  { id: 16, nombre: " Papas Fritas", precio: 3500, descripcion: "Porción clásica crujiente", categoria: "Acompañamientos", imagen: "https://okdiario.com/img/2023/04/12/el-truco-definitivo-para-que-las-patatas-fritas-te-queden-mas-crujientes.jpg" },
  { id: 17, nombre: " Aros de Cebolla", precio: 4500, descripcion: "Crujientes, salsa de ajo", categoria: "Acompañamientos", imagen: "https://es.cravingsjournal.com/wp-content/uploads/2022/05/aros-de-cebolla-1-500x375.jpg" },
  { id: 18, fontweight: " Ensalada", precio: 7500, descripcion: "Fresca con pollo grillado", categoria: "Acompañamientos", imagen: "https://mandolina.co/wp-content/uploads/2020/11/ensalada-de-pollo-aguacate-destacada-1200x720.jpg" },
  { id: 19, nombre: " Papas Criollas", precio: 4800, descripcion: "Con salsa de hogao", categoria: "Acompañamientos", imagen: "https://preparalapapa.com/wp-content/uploads/2020/09/PAPITAS-CRIOLLAS-ASADAS-CON-PAPRIKA-Y-ROMERO-1.png" },
  { id: 20, nombre: " Nuggets (8pz)", precio: 7500, descripcion: "Pollo empanizado con miel mostaza", categoria: "Acompañamientos", imagen: "https://images.unsplash.com/photo-1562967914-608f82629710?w=200&h=200&fit=crop" },
  { id: 21, nombre: " Gaseosa", precio: 1800, descripcion: "Personal 400ml", categoria: "Bebidas", imagen: "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTGPXlkwv9bIXhPilhBt8zc8XNZBXE3enO8Dw&s" },
  { id: 22, nombre: " Jugo Natural", precio: 2500, descripcion: "Fresa/Mora/Maracuyá", categoria: "Bebidas", imagen: "https://media-cdn.tripadvisor.com/media/photo-s/19/6f/f1/96/jugos-naturales-fresa.jpg" },
  { id: 23, nombre: " Sodas", precio: 6800, descripcion: "Frutos Rojos o Frutos Amarillos", categoria: "Bebidas", imagen: "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcT4vPvYnkjQYbtxt69-3d6c3AZVPSV5x3keJw&s" },
  { id: 24, nombre: " Limonadas", precio: 7200, descripcion: "Limonada natural, Cerezada, Limonada de panela", categoria: "Bebidas", imagen: "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcT4wJRRJY1PfMMg0Wn2gBMaP8FpGCZ43GRKig&s" },
  { id: 25, nombre: " Té Helado", precio: 3200, descripcion: "Durazno o limón", categoria: "Bebidas", imagen: "https://images.unsplash.com/photo-1556679343-c7306c1976bc?w=200&h=200&fit=crop" },
  { id: 26, nombre: " Alitas BBQ (6)", precio: 8900, descripcion: "Salsa BBQ y ranch", categoria: "Especialidades", imagen: "https://www.unileverfoodsolutions.com.co/dam/global-ufs/mcos/NOLA/calcmenu/recipes/col-recipies/fruco/ALITAS-SALSA-1024X1024-px.jpg" },
  { id: 27, nombre: " Alitas Buffalo", precio: 9200, descripcion: "Picantes, salsa azul", categoria: "Especialidades", imagen: "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTa8HRKDMKr9M9ZkURAZhP6tt8TmvRZJhKtoA&s" },
  { id: 28, nombre: " Pizza Personal", precio: 10500, descripcion: "Pepperoni o jamón", categoria: "Especialidades", imagen: "https://www.tillamook.com/_next/image?url=https%3A%2F%2Fimages.ctfassets.net%2Fj8tkpy1gjhi5%2F5OvVmigx6VIUsyoKz1EHUs%2Fb8173b7dcfbd6da341ce11bcebfa86ea%2FSalami-pizza-hero.jpg&w=3840&q=75" },
  { id: 29, nombre: " Tacos (3pz)", precio: 9900, descripcion: "Carne desmechada", categoria: "Especialidades", imagen: "https://img.lalr.co/cms/2023/01/30170540/canva-5.png?size=sm&ratio=sq&f=jpg" },
  { id: 30, nombre: " Picada", precio: 18500, descripcion: "Carne asada, chorizo, chicharrón, papa criolla, arepa", categoria: "Especialidades", imagen: "https://randys.com.co/wp-content/uploads/2020/12/Picada-2025.jpg" },
]);

function obtenerProductosFiltrados() {
  if (categoriaActiva.value === "Todos") return productos.value;
  return productos.value.filter(p => p.categoria === categoriaActiva.value);
}

function obtenerTotal() {
  return pedidoItems.value.reduce((total, item) => total + (item.precio * item.cantidad), 0);
}

function agregarAlPedido(producto) {
  let existente = pedidoItems.value.find(i => i.id === producto.id);
  if (existente) {
    existente.cantidad++;
  } else {
    pedidoItems.value.push({ ...producto, cantidad: 1 });
  }
  mostrarMensaje(`${producto.nombre} añadido!`);
}

function cambiarCantidad(item, delta) {
  const nueva = item.cantidad + delta;
  if (nueva <= 0) eliminarItem(item);
  else item.cantidad = nueva;
}

function eliminarItem(item) {
  pedidoItems.value = pedidoItems.value.filter(i => i !== item);
  mostrarMensaje("Eliminado 🗑️");
}

function limpiarPedido() {
  pedidoItems.value = [];
  mostrarMensaje("CARRITO VACÍO");
}

function enviarPedido() {
  if (pedidoItems.value.length === 0) return mostrarMensaje("Agrega algo primero!");

  itemsParaFactura.value = [...pedidoItems.value];
  totalFactura.value = obtenerTotal();
  nuevaFecha.value = new Date().toLocaleString();

  mostrarMensaje(`¡MESA ${numeroMesa.value}, TU PEDIDO VA EN CAMINO! 🚀`);
  
  setTimeout(() => {
    const elementoFactura = document.getElementById("factura-pdf");
    
    const opciones = {
      margin:       10,
      filename:     `Factura_Mesa_${numeroMesa.value}.pdf`,
      image:        { type: 'jpeg', quality: 0.98 },
      html2canvas:  { scale: 2, logging: false },
      jsPDF:        { unit: 'mm', format: 'a4', orientation: 'portrait' }
    };

    html2pdf().set(opciones).from(elementoFactura).save();
  }, 150);

  pedidoItems.value = [];
  mostrarCarrito.value = false;
}

function mostrarMensaje(mensaje) {
  mensajeAlerta.value = mensaje;
  mostrarAlerta.value = true;
  setTimeout(() => { mostrarAlerta.value = false; }, 2000);
}
</script>

<style scoped>
:root {
  --neon-yellow: #ccff00;
  --hot-red: #ff0033;
  --deep-black: #202020;
  --white: #ffffff;
}

.app {
  background: var(--deep-black);
  min-height: 100vh;
  padding: 20px;
  font-family: 'Poppins', sans-serif;
  color: var(--white);
}

.header {
  background: var(--hot-red);
  padding: 20px 30px;
  border-radius: 0 0 40px 40px;
  margin: -20px -20px 30px -20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  box-shadow: 0 10px 30px rgba(255, 0, 51, 0.3);
}

.logo-text { font-weight: 900; font-size: 2rem; letter-spacing: -1px; margin: 0; }
.logo-text span { color: var(--neon-yellow); }

.cart-btn-toggle {
  background: var(--neon-yellow);
  color: var(--deep-black);
  border: none;
  padding: 12px 25px;
  border-radius: 50px;
  font-weight: 900;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 10px;
}

.cart-badge { background: var(--hot-red); color: white; padding: 2px 8px; border-radius: 10px; }

.section-title {
  text-align: center;
  font-weight: 900;
  font-size: 2.5rem;
  color: var(--neon-yellow);
  margin-bottom: 30px;
  text-shadow: 3px 3px 0px var(--hot-red);
}

.categorias { display: flex; gap: 10px; overflow-x: auto; padding: 10px 0 25px 0; }

.cat-btn {
  background: #1d1d1d;
  color: #f1ebeb;
  border: 2px solid #333;
  padding: 12px 25px;
  border-radius: 15px;
  font-weight: 700;
  cursor: pointer;
}

.cat-btn.active { background: var(--neon-yellow); color: var(--deep-black); border-color: var(--neon-yellow); }

.productos-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(240px, 1fr)); gap: 30px; }

.producto-card {
  background: #eceaea;
  border-radius: 30px;
  overflow: hidden;
  border: 2px solid #161616;
  transition: 0.3s;
  color: #1a1a1a;
}

.producto-card:hover { border-color: var(--hot-red); transform: translateY(-10px); }

.producto-imagen { height: 180px; background-size: cover; background-position: center; }

.producto-info { padding: 20px; display: flex; flex-direction: column; gap: 5px; }

.producto-nombre { font-size: 1.2rem; font-weight: 900; color: #111; }

.descripcion { color: #666; font-size: 0.85rem; min-height: 40px; }

.precio-inline {
  font-size: 1.4rem;
  font-weight: 900;
  color: var(--hot-red);
  margin: 5px 0;
}

.add-btn-style {
  background: var(--neon-yellow);
  color: var(--deep-black);
  padding: 12px;
  border-radius: 15px;
  text-align: center;
  font-weight: 900;
  font-size: 0.9rem;
  cursor: pointer;
  margin-top: 5px;
}

.modal-overlay {
  position: fixed;
  top: 0; left: 0; width: 100%; height: 100%;
  background: rgba(134, 134, 134, 0.85);
  display: flex; justify-content: center; align-items: center;
  z-index: 1000;
}

.modal-content {
  background: #181717;
  width: 95%;
  max-width: 500px;
  border: 4px solid var(--hot-red);
  border-radius: 40px;
  padding: 30px;
  color: rgb(245, 243, 243);
}

.item-pedido {
  display: flex;
  align-items: center;
  background: #131212;
  padding: 15px;
  border-radius: 20px;
  margin-bottom: 10px;
}

.item-miniatura { width: 50px; height: 50px; border-radius: 10px; background-size: cover; margin-right: 15px; }

.order-total { font-size: 2rem; font-weight: 900; text-align: center; margin: 20px 0; }
.total-monto { color: var(--neon-yellow); }

.btn-enviar {
  background: var(--hot-red);
  color: white;
  padding: 18px;
  border-radius: 20px;
  width: 100%;
  border: none;
  font-weight: 900;
  cursor: pointer;
}

.alert-box {
  position: fixed;
  top: 40px;
  left: 50%;
  transform: translateX(-50%);
  background: var(--neon-yellow);
  color: var(--deep-black);
  padding: 15px 40px;
  border-radius: 50px;
  font-weight: 900;
  z-index: 2000;
}

.factura-container {
  font-family: 'Poppins', sans-serif;
  color: #222222;
  background: #ffffff;
  padding: 30px;
  max-width: 650px;
  margin: 0 auto;
}

.factura-header {
  text-align: center;
  border-bottom: 3px double #222;
  padding-bottom: 15px;
  margin-bottom: 20px;
}

.factura-header h2 {
  font-weight: 900;
  font-size: 2.2rem;
  color: #ff0033; 
  margin: 0;
}

.factura-header p {
  font-size: 0.9rem;
  color: #666;
  margin: 5px 0 0 0;
}

.factura-meta {
  margin-bottom: 25px;
  font-size: 0.95rem;
  line-height: 1.5;
}

.factura-meta p { margin: 4px 0; }

.factura-tabla {
  width: 100%;
  border-collapse: collapse;
  margin-bottom: 25px;
}

.factura-tabla th {
  background: #f4f4f4;
  text-align: left;
  padding: 10px;
  font-weight: 700;
  border-bottom: 2px solid #222;
}

.factura-tabla td {
  padding: 12px 10px;
  border-bottom: 1px solid #eee;
  font-size: 0.95rem;
}

.factura-total {
  text-align: right;
  font-size: 1.4rem;
  border-top: 2px solid #222;
  padding-top: 15px;
  margin-bottom: 35px;
}

.factura-total span {
  font-weight: 400;
  margin-right: 15px;
}

.factura-total strong {
  font-weight: 900;
  color: #ff0033;
}

.factura-footer {
  text-align: center;
  font-size: 0.85rem;
  color: #777;
  border-top: 1px dashed #ccc;
  padding-top: 15px;
}
.factura-footer p { margin: 3px 0; }

@media (max-width: 600px) {
  .productos-grid { grid-template-columns: 1fr 1fr; gap: 15px; }
  .producto-nombre { font-size: 1rem; }
  .precio-inline { font-size: 1.1rem; }
}
</style>
