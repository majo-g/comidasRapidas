<template>
  <link href="https://fonts.googleapis.com/css2?family=Poppins:wght=400;600;700;900&display=swap" rel="stylesheet">
  
  <div class="app">
    <header class="header">
      <h1 class="logo-text">RAPIDO <span>&</span> RICO</h1>
      <div class="header-right">
        <transition name="fade">
          <span v-if="notificacionTexto" class="notificacion-header">// {{ notificacionTexto }}</span>
        </transition>
        
        <button class="btn-toggle-admin" v-on:click="alternarFormulario">
          {{ editandoId ? '✏️ Editando...' : '⚙️ Menú Admin' }}
        </button>
        <div class="mesa-info"> {{ productos.length }} Productos</div>
        
        <button 
          class="cart-btn-toggle" 
          :class="{ 'animar-pulso': animarCarrito }" 
          v-on:click="mostrarCarrito = true"
        >
          🛒
          <span class="cart-badge" v-if="pedidoItems.length > 0">{{ pedidoItems.length }}</span>
        </button>
      </div>
    </header>

    <main class="dashboard">
      <transition name="slide">
        <section v-if="mostrarFormulario" class="admin-panel-right">
          <div class="panel-header">
            <h3>{{ editandoId ? 'Editar Producto' : 'Nuevo Producto' }}</h3>
            <button class="close-panel" v-on:click="limpiarFormulario">✕</button>
          </div>

          <!-- Agregamos novalidate para apagar por completo las alertas nativas del navegador -->
          <form v-on:submit.prevent="guardarProducto" class="panel-form" novalidate>

            <div class="form-group">
              <label class="form-label-white">Nombre del Producto:</label>
              <input type="text" v-model="nuevoProducto.nombre" placeholder="Ej. Hamburguesa Triple" class="input-dark" />
            </div>

            <div class="form-group">
              <label class="form-label-white">Precio (COP):</label>
              <input type="number" v-model.number="nuevoProducto.precio" placeholder="Ej. 15000" min="0" class="input-dark" />
            </div>

            <div class="form-group">
              <label class="form-label-white">Categoría:</label>
              <select v-model="nuevoProducto.categoria" class="input-dark">
                <option v-for="cat in categoriasDisponibles" :key="cat" :value="cat">
                  {{ cat }}
                </option>
              </select>
            </div>

            <div class="form-group">
              <label class="form-label-white">Descripción:</label>
              <input type="text" v-model="nuevoProducto.descripcion" placeholder="Ej. Triple carne, queso cheddar..." class="input-dark" />
            </div>

            <div class="form-group">
              <label class="form-label-white">URL de la Imagen:</label>
              <input type="url" v-model="nuevoProducto.imagen" placeholder="https://ejemplo.com/imagen.jpg" class="input-dark" />
            </div>

            <button type="submit" class="btn-guardar-producto">
              {{ editandoId ? 'Actualizar Cambios 💾' : 'Guardar en Menú 💾' }}
            </button>
          </form>
        </section>
      </transition>

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
          >
            <div v-on:click="agregarAlPedido(p)">
              <div class="producto-imagen" :style="{ backgroundImage: 'url(' + p.imagen + ')' }"></div>
              
              <div class="producto-info">
                <strong class="producto-nombre">{{ p.nombre }}</strong>
                <small class="descripcion">{{ p.descripcion }}</small>
                <div class="precio-inline">{{ formatearMoneda(p.precio) }}</div>
                <div class="add-btn-style">Agregar Al Carrito</div>
              </div>
            </div>

            <div class="admin-actions-bar">
              <button class="btn-action-edit" v-on:click="cargarEdicion(p)">✏️ Editar</button>
              
              <button 
                class="btn-action-delete" 
                :class="{ 'esperando-confirmacion': idConfirmacionEliminar === p.id }"
                v-on:click="manejarEliminacionMenu(p)"
              >
                {{ idConfirmacionEliminar === p.id ? '¿Seguro? ⚠️' : '🗑️ Eliminar' }}
              </button>
            </div>
          </div>
        </div>
      </section>

      <div class="modal-overlay" v-if="mostrarCarrito" v-on:click.self="mostrarCarrito = false">
        <section class="modal-content">
          <div class="order-header">
            <h2 class="modal-title">Mi Pedido 🍔</h2>
            <button class="close-modal" v-on:click="mostrarCarrito = false">✕</button>
          </div>
          
          <div class="mesa-selector">
            <label>Número de Mesa:</label>
            <div class="input-wrapper">
              <span>📍</span>
              <input type="number" v-model="numeroMesa" min="1" max="99" />
            </div>
          </div>

          <div class="lista-pedido" v-if="pedidoItems.length > 0">
            <div v-for="(item, idx) in pedidoItems" :key="idx" class="item-pedido">
              <div class="item-miniatura" :style="{ backgroundImage: 'url(' + item.imagen + ')' }"></div>
              
              <div class="item-detalles">
                <span class="item-nombre">{{ item.nombre }}</span>
                <span class="item-subtotal">{{ formatearMoneda(item.precio * item.cantidad) }}</span>
              </div>
              
              <div class="item-acciones">
                <div class="cantidad-box">
                  <button class="qty-btn" v-on:click="cambiarCantidad(item, -1)">-</button>
                  <span class="qty-num">{{ item.cantidad }}</span>
                  <button class="qty-btn" v-on:click="cambiarCantidad(item, 1)">+</button>
                </div>
                <button class="del-btn" v-on:click="eliminarItem(item)" title="Eliminar producto">🗑️</button>
              </div>
            </div>
          </div>

          <div class="pedido-vacio" v-else>
            <div class="vacio-icono">🛒</div>
            <p>Tu carrito está vacío</p>
            <span>¡Agrega tus platillos favoritos para empezar!</span>
          </div>

          <div class="modal-footer" v-if="pedidoItems.length > 0">
            <div class="order-total">
              <span class="total-label">Total a pagar:</span>
              <span class="total-monto">{{ formatearMoneda(obtenerTotal()) }}</span>
            </div>

            <div class="btn-group">
              <button class="btn btn-enviar" v-on:click="enviarPedido">CONFIRMAR PEDIDO
</button>
              <button class="btn btn-limpiar" v-on:click="limpiarPedido">Vaciar carrito</button>
            </div>
          </div>
        </section>
      </div>
    </main>

    <div class="factura-wrapper-oculto">
      <div id="factura-pdf" class="factura-container">
        <div class="factura-header">
          <h2>RÁPIDO & RICO</h2>
          <p class="factura-subtitle">Comprobante de Pedido Digital</p>
        </div>
        
        <div class="factura-meta">
          <div class="meta-item"><strong>Fecha y Hora:</strong> <span>{{ nuevaFecha }}</span></div>
          <div class="meta-item"><strong>Ubicación:</strong> <span>Mesa {{ numeroMesa }}</span></div>
          <div class="meta-item"><strong>Estado:</strong> <span class="badge-status">Confirmado en Cocina </span></div>
        </div>

        <table class="factura-tabla">
          <thead>
            <tr>
              <th style="width: 15%; text-align: center;">Cant.</th>
              <th style="width: 50%;">Producto</th>
              <th style="width: 15%; text-align: right;">Unitario</th>
              <th style="width: 20%; text-align: right;">Subtotal</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in itemsParaFactura" :key="item.id">
              <td style="text-align: center; font-weight: 600;">{{ item.cantidad }}x</td>
              <td class="factura-item-nombre">{{ item.nombre }}</td>
              <td style="text-align: right;">{{ formatearMoneda(item.precio) }}</td>
              <td style="text-align: right; font-weight: 600;">{{ formatearMoneda(item.precio * item.cantidad) }}</td>
            </tr>
          </tbody>
        </table>

        <div class="factura-total-block">
          <div class="total-row">
            <span class="total-block-label">TOTAL A PAGAR</span>
            <strong class="total-block-monto">{{ formatearMoneda(totalFactura) }}</strong>
          </div>
        </div>

        <div class="factura-footer">
          <p class="gracias-msg">¡Muchas gracias por elegirnos!</p>
          <p>Tu orden fue enviada y se encuentra actualmente en preparación.</p>
          <p class="ticket-id">Verifica tu pedido con el personal del restaurante.</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from "vue";
import Swal from 'https://cdn.jsdelivr.net/npm/sweetalert2@11/+esm'

const categoriaActiva = ref("Todos");
const numeroMesa = ref(1);
const pedidoItems = ref([]);
const mostrarCarrito = ref(false);

const mostrarFormulario = ref(false);
const editandoId = ref(null); 
const nuevoProducto = ref({ nombre: "", precio: null, descripcion: "", categoria: "Hamburguesas", imagen: "" });

// Estados de control interno
const notificacionTexto = ref("");
const alertaFormulario = ref(""); // <--- NUEVO: Controla el texto del error en el panel admin
const idConfirmacionEliminar = ref(null);
const animarCarrito = ref(false); 
let temporizadorConfirmacion = null;

const itemsParaFactura = ref([]);
const totalFactura = ref(0);
const nuevaFecha = ref("");

const categoriesString = ["Todos", "Hamburguesas", "Perros", "Salchipapas", "Acompañamientos", "Bebidas", "Especialidades"];
const categorias = ref(categoriesString);

const categoriasDisponibles = computed(() => {
  return categorias.value.filter(c => c !== "Todos");
});

const productos = ref([
  { id: 1, nombre: " Hamburguesa Simple", precio: 10500, descripcion: "Carne 150g, lechuga, tomate", categoria: "Hamburguesas", imagen: "https://img.hogar.mapfre.es/wp-content/uploads/2018/09/hamburguesa-sencilla.jpg" },
  { id: 2, nombre: " Hamburguesa Doble", precio: 13000, descripcion: "Doble carne, queso cheddar", categoria: "Hamburguesas", imagen: "https://thumbs.dreamstime.com/b/burger-doble-con-tocino-en-placa-de-pizarra-hamburguesa-dos-patatas-ternera-queso-cheddar-crujiente-y-lechuga-verde-fresca-388435375.jpg" },
  { id: 3, nombre: " Hamburguesa BBQ", precio: 16500, descripcion: "Salsa BBQ, cebolla crispy", categoria: "Hamburguesas", imagen: "https://www.shutterstock.com/image-photo/beef-burger-bbq-sauce-bacon-600nw-2471612047.jpg" },
  { id: 4, nombre: " Hamburguesa Hawaiana", precio: 15000, descripcion: "Piña, jamón, queso", categoria: "Hamburguesas", imagen: "https://thumbs.dreamstime.com/b/hamburguesa-hawaiana-con-la-pi%C3%B1a-el-jam%C3%B3n-y-queso-107562948.jpg" },
  { id: 5, nombre: " Hamburguesa Mexicana", precio: 18500, descripcion: "Guacamole, jalapeños", categoria: "Hamburguesas", imagen: "https://www.vvsupremo.com/wp-content/uploads/2018/05/Mexican-Burger-with-Chorizo.jpg" },
  { id: 6, nombre: " Perro Caliente", precio: 5000, descripcion: "Salchicha, papitas, salsas", categoria: "Perros", imagen: "https://media.istockphoto.com/id/1161043951/es/foto/perro-colombiano-paisa.jpg?s=612x612&w=0&k=20&c=-5Xi0DKEPupdcMSDJoymfhCIDVEgAjbHxohGu3ldwfE=" },
  { id: 7, nombre: " Perro Especial", precio: 8500, descripcion: "Tocineta, queso, piña", categoria: "Perros", imagen: "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcRNiaRNJ1ZXzBSvcjB9jpBhpzqJEkK8-6htGg&s" },
  { id: 8, fontAwesome: " Perro Americano", nombre: " Perro Americano", precio: 10500, descripcion: "Chili con carne, queso", categoria: "Perros", imagen: "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTF8UbHxjp_hq5NviYymYMk1Tk4Ar9EFX6__A&s" },
  { id: 9, nombre: " Perro Ranchero", precio: 12000, descripcion: "Maíz, mayonesa chipotle", categoria: "Perros", imagen: "https://0201.nccdn.net/1_2/000/000/192/ea4/perro-ranchero.jpg" },
  { id: 10, nombre: " Perro Colombiano", precio: 10500, descripcion: "Piña, papa criolla", categoria: "Perros", imagen: "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTLxDX474LGREwRLHZfkQAEYVZVWQSBewxU6w&s" },
  { id: 11, nombre: " Salchipapa Clásica", precio: 9500, descripcion: "Papas fritas, salchicha, salsa rosada", categoria: "Salchipapas", imagen: "https://imag.bonviveur.com/emplatado-final-de-las-salchipapas.jpg" },
  { id: 12, nombre: " Salchipapa Queso", precio: 9500, descripcion: "Papas, salchicha, queso mozzarella gratinado", categoria: "Salchipapas", imagen: "https://lariendagastrofonda.com/wp-content/uploads/2023/08/salchipapas.png" },
  { id: 13, nombre: " Salchipapa Especial", precio: 14500, descripcion: "Papas, salchicha, tocineta, chorizo", categoria: "Salchipapas", imagen: "https://cloudfront-us-east-1.images.arcpublishing.com/infobae/CRU7IWBQBVBWTDZT2AZFIXREVI.jpg" },
  { id: 14, nombre: " Salchipapa Hawaiana", precio: 12000, descripcion: "Papas, salchicha, piña, queso", categoria: "Salchipapas", imagen: "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcSWV0MgnQr5PxvSyj1xBeQiNI27p1bKJ-BASQ&s" },
  { id: 15, nombre: " Salchipapa BBQ", precio: 16000, descripcion: "Papas, salchicha, salsa BBQ, pollo desmechado", categoria: "Salchipapas", imagen: "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcRxrFNEnW34VJOgCtM4GKk7T_ovlf7ICchWQw&s" },
  { id: 16, nombre: " Papas Fritas", precio: 3500, descripcion: "Porción clásica crujiente", categoria: "Acompañamientos", imagen: "https://okdiario.com/img/2023/04/12/el-truco-definitivo-para-que-las-patatas-fritas-te-queden-mas-crujientes.jpg" },
  { id: 17, nombre: " Aros de Cebolla", precio: 4500, descripcion: "Crujientes, salsa de ajo", categoria: "Acompañamientos", imagen: "https://es.cravingsjournal.com/wp-content/uploads/2022/05/aros-de-cebolla-1-500x375.jpg" },
  { id: 18, nombre: " Ensalada", precio: 7500, descripcion: "Fresca con pollo grillado", categoria: "Acompañamientos", imagen: "https://mandolina.co/wp-content/uploads/2020/11/ensalada-de-pollo-aguacate-destacada-1200x720.jpg" },
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

function formatearMoneda(valor) {
  return new Intl.NumberFormat('es-CO', { style: 'currency', currency: 'COP', minimumFractionDigits: 0 }).format(valor);
}

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
  
  // ALERTA TIPO TOAST CON SWEETALERT2
  Swal.fire({
    icon: "success",
    title: "¡Agregado al carrito!",
    text: `${producto.nombre.trim()} se sumó a tu pedido.`,
    toast: true,                
    position: "top-end",        
    showConfirmButton: false,   
    timer: 2000,                
    timerProgressBar: true,     
    background: "#1a1a1a",      
    color: "#ffffff"            
  });
  
  // Mantiene tu animación de pulso en el botón flotante del carrito
  animarCarrito.value = true;
  setTimeout(() => {
    animarCarrito.value = false;
  }, 400); 
}

// NUEVA VALIDACIÓN CONTROLADA DESDE VUE (Sin alertas HTML toscas)
function guardarProducto() {
  // Validaciones manuales usando la variable Swal directamente
  if (!nuevoProducto.value.nombre || nuevoProducto.value.nombre.trim() === "") {
    Swal.fire({
      icon: "error",
      title: "¡Faltan datos!",
      text: "Por favor ingresa el nombre del producto.",
      confirmButtonColor: "#ff0033"
    });
    return;
  }
  
  if (nuevoProducto.value.precio === null || nuevoProducto.value.precio === undefined || nuevoProducto.value.precio < 0) {
    Swal.fire({
      icon: "error",
      title: "¡Precio inválido!",
      text: "Por favor ingresa un precio válido (mayor o igual a 0).",
      confirmButtonColor: "#ff0033"
    });
    return;
  }
  
  if (!nuevoProducto.value.descripcion || nuevoProducto.value.descripcion.trim() === "") {
    Swal.fire({
      icon: "error",
      title: "¡Faltan datos!",
      text: "Por favor agrega una breve descripción.",
      confirmButtonColor: "#ff0033"
    });
    return;
  }
  
  if (!nuevoProducto.value.imagen || nuevoProducto.value.imagen.trim() === "") {
    Swal.fire({
      icon: "error",
      title: "¡Faltan datos!",
      text: "Por favor pega un enlace o URL de imagen válido.",
      confirmButtonColor: "#ff0033"
    });
    return;
  }

  // Si pasa el filtro, procesamos el guardado
  if (editandoId.value !== null) {
    const indice = productos.value.findIndex(p => p.id === editandoId.value);
    if (indice !== -1) {
      let nombreFormateado = nuevoProducto.value.nombre;
      if (!nombreFormateado.startsWith(" ")) nombreFormateado = " " + nombreFormateado;

      productos.value[indice] = {
        id: editandoId.value,
        nombre: nombreFormateado,
        precio: nuevoProducto.value.precio,
        descripcion: nuevoProducto.value.descripcion,
        categoria: nuevoProducto.value.categoria,
        imagen: nuevoProducto.value.imagen
      };
      
      Swal.fire({
        icon: "success",
        title: "¡Actualizado!",
        text: `${nuevoProducto.value.nombre.trim()} se modificó correctamente.`,
        timer: 2000,
        showConfirmButton: false
      });
    }
  } else {
    const nuevoId = productos.value.length > 0 ? Math.max(...productos.value.map(p => p.id)) + 1 : 1;
    
    productos.value.push({
      id: nuevoId,
      nombre: " " + nuevoProducto.value.nombre, 
      precio: nuevoProducto.value.precio,
      descripcion: nuevoProducto.value.descripcion,
      categoria: nuevoProducto.value.categoria,
      imagen: nuevoProducto.value.imagen
    });

    Swal.fire({
      icon: "success",
      title: "¡Creado con éxito!",
      text: `${nuevoProducto.value.nombre.trim()} fue agregado al menú.`,
      timer: 2000,
      showConfirmButton: false
    });
  }
  limpiarFormulario();
}

function desplegarAlertaFormulario(msg) {
  alertaFormulario.value = msg;
  setTimeout(() => {
    if (alertaFormulario.value === msg) alertaFormulario.value = "";
  }, 4000);
}

function cargarEdicion(producto) {
  editandoId.value = producto.id;
  nuevoProducto.value = {
    nombre: producto.nombre.trim(), 
    precio: producto.precio,
    descripcion: producto.descripcion,
    categoria: producto.categoria,
    imagen: producto.imagen
  };
  mostrarFormulario.value = true;
}

function manejarEliminacionMenu(producto) {
  if (idConfirmacionEliminar.value === producto.id) {
    productos.value = productos.value.filter(p => p.id !== producto.id);
    pedidoItems.value = pedidoItems.value.filter(i => i !== producto);
    ejecutarNotificacionInterna("Menú actualizado (Eliminado)");
    
    if (editandoId.value === producto.id) limpiarFormulario();
    idConfirmacionEliminar.value = null;
    clearTimeout(temporizadorConfirmacion);
  } else {
    idConfirmacionEliminar.value = producto.id;
    clearTimeout(temporizadorConfirmacion);
    temporizadorConfirmacion = setTimeout(() => {
      idConfirmacionEliminar.value = null;
    }, 3000);
  }
}

function alternarFormulario() {
  if (mostrarFormulario.value && editandoId.value === null) {
    limpiarFormulario();
  } else {
    mostrarFormulario.value = true;
  }
}

function limpiarFormulario() {
  nuevoProducto.value = { nombre: "", precio: null, descripcion: "", categoria: "Hamburguesas", imagen: "" };
  editandoId.value = null;
  mostrarFormulario.value = false;
  alertaFormulario.value = "";
}

function cambiarCantidad(item, delta) {
  const nueva = item.cantidad + delta;
  if (nueva <= 0) eliminarItem(item);
  else item.cantidad = nueva;
}

function eliminarItem(item) {
  pedidoItems.value = pedidoItems.value.filter(i => i !== item);
  ejecutarNotificacionInterna("Item removido");
}

function limpiarPedido() {
  pedidoItems.value = [];
  ejecutarNotificacionInterna("Carrito vacío");
}

function enviarPedido() {
  if (pedidoItems.value.length === 0) return;

  if (!window.jspdf) {
    ejecutarNotificacionInterna("jsPDF no está cargado");
    return;
  }

  const { jsPDF } = window.jspdf;
  const doc = new jsPDF();

  // ENCABEZADO
  doc.setFillColor(255, 0, 51);
  doc.rect(0, 0, 210, 35, "F");

  doc.setTextColor(255, 255, 255);
  doc.setFontSize(24);
  doc.setFont("helvetica", "bold");
  doc.text("RAPIDO & RICO", 105, 18, { align: "center" });

  doc.setFontSize(10);
  doc.text("COMPROBANTE OFICIAL DE PEDIDO", 105, 27, {
    align: "center",
  });

  // DATOS
  let y = 50;

  doc.setTextColor(0, 0, 0);
  doc.setFontSize(12);

  doc.text(`Mesa: ${numeroMesa.value}`, 15, y);
  y += 8;

  doc.text(
    `Fecha: ${new Date().toLocaleDateString()}`,
    15,
    y
  );
  y += 8;

  doc.text(
    `Hora: ${new Date().toLocaleTimeString()}`,
    15,
    y
  );

  // TABLA
  y += 15;

  doc.setFillColor(30, 30, 30);
  doc.rect(15, y - 6, 180, 10, "F");

  doc.setTextColor(255, 255, 255);
  doc.setFontSize(11);

  doc.text("PRODUCTO", 20, y);
  doc.text("CANT", 120, y);
  doc.text("SUBTOTAL", 190, y, { align: "right" });

  y += 12;

  doc.setTextColor(0, 0, 0);

  pedidoItems.value.forEach(item => {
    const subtotal = item.precio * item.cantidad;

    doc.setFontSize(10);

    doc.text(item.nombre.trim(), 20, y);

    doc.text(
      item.cantidad.toString(),
      125,
      y
    );

    doc.text(
      formatearMoneda(subtotal),
      190,
      y,
      { align: "right" }
    );

    y += 10;

    doc.setDrawColor(230, 230, 230);
    doc.line(15, y - 4, 195, y - 4);

    if (y > 250) {
      doc.addPage();
      y = 20;
    }
  });

  // TOTAL
  y += 10;

  doc.setFillColor(204, 255, 0);
  doc.roundedRect(120, y - 5, 75, 18, 3, 3, "F");

  doc.setFontSize(12);
  doc.setFont("helvetica", "bold");

  doc.text(
    "TOTAL:",
    130,
    y + 6
  );

  doc.text(
    formatearMoneda(obtenerTotal()),
    190,
    y + 6,
    { align: "right" }
  );

  // PIE
  y += 35;

  doc.setDrawColor(180);
  doc.line(20, y, 190, y);

  y += 12;

  doc.setFontSize(10);
  doc.setFont("helvetica", "normal");

  doc.text(
    "Gracias por preferir RAPIDO & RICO",
    105,
    y,
    { align: "center" }
  );

  y += 8;

  doc.text(
    "Tu pedido fue enviado correctamente a cocina.",
    105,
    y,
    { align: "center" }
  );

  y += 8;

  doc.text(
    "Presenta este comprobante al momento de pagar.",
    105,
    y,
    { align: "center" }
  );

  doc.save(`Factura_Mesa_${numeroMesa.value}.pdf`);

  mostrarCarrito.value = false;

  ejecutarNotificacionInterna("PDF generado correctamente");
}

function ejecutarNotificacionInterna(mensaje) {
  notificacionTexto.value = mensaje;
  setTimeout(() => {
    if (notificacionTexto.value === mensaje) notificacionTexto.value = "";
  }, 3000);
}
</script>

<style scoped>
:root {
  --neon-yellow: #ccff00;
  --hot-red: #ff0033;
  --deep-black: #202020;
  --white: #ffffff;
}

/* NUEVA CLASE: Alerta integrada dentro del panel del Administrador */
.alerta-formulario-interna {
  background: rgba(255, 0, 51, 0.15);
  color: #ff3355;
  border: 1px solid var(--hot-red);
  padding: 12px;
  border-radius: 10px;
  font-size: 0.85rem;
  font-weight: 600;
  margin-bottom: 20px;
  line-height: 1.4;
  text-align: left;
}

.notificacion-header {
  font-size: 0.9rem;
  font-weight: 700;
  color: var(--neon-yellow);
  background: rgba(0, 0, 0, 0.4);
  padding: 6px 14px;
  border-radius: 8px;
  border-left: 3px solid var(--neon-yellow);
}

.fade-enter-active, .fade-leave-active { transition: opacity 0.2s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }

.cart-btn-toggle.animar-pulso {
  animation: saltoPop 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}

@keyframes saltoPop {
  0% { transform: scale(1); }
  50% { transform: scale(1.3); background: #ffffff; box-shadow: 0 0 25px var(--neon-yellow); }
  100% { transform: scale(1); }
}

.btn-action-delete.esperando-confirmacion {
  background: #ff0033 !important;
  color: #ffffff !important;
  animation: pulsoAdvertencia 1s infinite alternate;
}

@keyframes pulsoAdvertencia {
  from { opacity: 0.8; }
  to { opacity: 1; box-shadow: 0 0 8px #ff0033; }
}

.factura-wrapper-oculto {
  position: absolute;
  left: -9999px;
  top: -9999px;
  width: 550px;
  overflow: hidden;
}

.app {
  background: var(--deep-black);
  min-height: 100vh;
  padding: 20px;
  font-family: 'Poppins', sans-serif;
  color: var(--white);
  overflow-x: hidden; 
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

.header-right { display: flex; align-items: center; gap: 15px; }
.mesa-info { background: rgba(0, 0, 0, 0.2); padding: 8px 16px; border-radius: 20px; font-weight: 600; font-size: 0.95rem; }

.btn-toggle-admin {
  background: var(--deep-black);
  color: var(--white);
  border: 2px solid var(--neon-yellow);
  padding: 10px 18px;
  border-radius: 14px;
  font-size: 0.9rem;
  font-weight: 700;
  cursor: pointer;
  transition: 0.2s;
}
.btn-toggle-admin:hover { 
  background: var(--neon-yellow);
  color: var(--deep-black);
  transform: scale(1.05);
}

.admin-panel-right {
  position: fixed;
  top: 0;
  right: 0;
  width: 350px;
  height: 100vh;
  background: #1a1a1a;
  border-left: 2px solid #333333;
  box-shadow: -10px 0 30px rgba(0, 0, 0, 0.5);
  z-index: 10005; 
  padding: 25px;
  display: flex;
  flex-direction: column;
  box-sizing: border-box;
}

.panel-header { display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #3d3d3d; padding-bottom: 15px; margin-bottom: 20px; }
.panel-header h3 { margin: 0; font-size: 1.4rem; font-weight: 700; color: #ffffff; }

.close-panel { background: #2d2d2d; color: #aaa; border: none; width: 32px; height: 32px; border-radius: 50%; cursor: pointer; font-weight: 700; }
.close-panel:hover { background: var(--hot-red); color: white; }

.panel-form { flex: 1; overflow-y: auto; padding-right: 5px; }

.slide-enter-active, .slide-leave-active { transition: transform 0.3s ease; }
.slide-enter-from, .slide-leave-to { transform: translateX(100%); }

.form-group { display: flex; flex-direction: column; gap: 6px; margin-bottom: 16px; }
.form-label-white { font-size: 0.85rem; font-weight: 600; color: #ffffff; }

.input-dark {
  background: #252525;
  border: 1px solid #444444;
  padding: 11px 14px;
  border-radius: 8px;
  color: #ffffff;
  font-family: 'Poppins', sans-serif;
  font-size: 0.9rem;
  outline: none;
  width: 100%;
  box-sizing: border-box;
}
.input-dark:focus { border-color: var(--neon-yellow); }

.btn-guardar-producto {
  width: 100%;
  background: var(--neon-yellow);
  color: var(--deep-black);
  border: none;
  padding: 14px;
  border-radius: 10px;
  font-weight: 900;
  font-size: 1rem;
  cursor: pointer;
  margin-top: 15px;
  box-shadow: 0 4px 15px rgba(204, 255, 0, 0.2);
}

.admin-actions-bar { display: flex; border-top: 1px solid #dddddd; background: #f8f8f8; padding: 8px; gap: 8px; }
.btn-action-edit, .btn-action-delete { flex: 1; border: none; padding: 8px; font-size: 0.85rem; font-weight: 700; border-radius: 8px; cursor: pointer; font-family: 'Poppins', sans-serif; transition: background 0.2s; }
.btn-action-edit { background: #e2ecff; color: #0055ff; }
.btn-action-delete { background: #ffe2e6; color: #ff0033; }

.card { background: transparent; }

.cart-btn-toggle {
  background: #ccff00 ; color: #000000 ; border: 3px solid #000000 ;
  width: 70px; height: 70px; border-radius: 50%; font-size: 2rem; cursor: pointer;
  display: flex; align-items: center; justify-content: center;
  position: fixed; bottom: 25px; right: 25px; z-index: 99999 ;
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.6) ; transition: transform 0.2s;
}
.cart-btn-toggle:hover { transform: scale(1.1); }
.cart-badge { position: absolute; top: -6px; right: -6px; background: #ff0033 ; color: #ffffff ; padding: 4px 10px; border-radius: 20px; font-size: 0.95rem; font-weight: 900; border: 2px solid #000000 ; }

.section-title { text-align: center; font-weight: 900; font-size: 2.5rem; color: var(--neon-yellow); margin-bottom: 30px; text-shadow: 3px 3px 0px var(--hot-red); }
.categorias { display: flex; gap: 10px; overflow-x: auto; padding: 10px 0 25px 0; }
.cat-btn { background: #1d1d1d; color: #f1ebeb; border: 2px solid #333; padding: 12px 25px; border-radius: 15px; font-weight: 700; cursor: pointer; }
.cat-btn.active { background: var(--neon-yellow); color: var(--deep-black); border-color: var(--neon-yellow); }

.productos-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(240px, 1fr)); gap: 30px; }
.producto-card { background: #eceaea; border-radius: 30px; overflow: hidden; border: 2px solid #161616; display: flex; flex-direction: column; justify-content: space-between; }
.producto-imagen { height: 180px; background-size: cover; background-position: center; cursor: pointer; }
.producto-info { padding: 20px; display: flex; flex-direction: column; gap: 5px; cursor: pointer; }
.producto-nombre { font-size: 1.2rem; font-weight: 900; color: #111; }
.descripcion { color: #666; font-size: 0.85rem; min-height: 40px; }
.precio-inline { font-size: 1.4rem; font-weight: 900; color: var(--hot-red); margin: 5px 0; }
.add-btn-style { background: var(--neon-yellow); color: var(--deep-black); padding: 12px; border-radius: 15px; text-align: center; font-weight: 900; font-size: 0.9rem; cursor: pointer; }

.modal-overlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0, 0, 0, 0.85); display: flex; justify-content: center; align-items: center; z-index: 10000; backdrop-filter: blur(4px); }
.modal-content { background: #1a1a1a; width: 92%; max-width: 480px; max-height: 85vh; border-radius: 24px; padding: 24px; color: #ffffff; display: flex; flex-direction: column; border: 1px solid #333333; }
.order-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; padding-bottom: 12px; border-bottom: 1px solid #2d2d2d; }
.modal-title { font-size: 1.6rem; font-weight: 700; margin: 0; }
.close-modal { background: #2d2d2d; color: #aaa; border: none; width: 36px; height: 36px; border-radius: 50%; cursor: pointer; display: flex; align-items: center; justify-content: center; }

.mesa-selector { background: #222222; padding: 12px 16px; border-radius: 14px; display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
.input-wrapper { display: flex; align-items: center; background: #1a1a1a; border: 1px solid #444; border-radius: 8px; padding: 4px 10px; }
.input-wrapper input { background: transparent; border: none; color: var(--neon-yellow); font-weight: 700; font-size: 1.1rem; width: 50px; text-align: center; outline: none; }

.lista-pedido { flex: 1; overflow-y: auto; margin-bottom: 15px; }
.item-pedido { display: flex; align-items: center; background: #222222; padding: 12px; border-radius: 16px; margin-bottom: 12px; border: 1px solid #2d2d2d; }
.item-miniatura { width: 55px; height: 55px; border-radius: 12px; background-size: cover; background-position: center; flex-shrink: 0; }
.item-detalles { flex: 1; padding: 0 12px; display: flex; flex-direction: column; gap: 4px; }
.item-subtotal { font-size: 1rem; font-weight: 700; color: var(--neon-yellow); }
.item-acciones { display: flex; align-items: center; gap: 12px; }
.cantidad-box { display: flex; align-items: center; background: #1a1a1a; border-radius: 10px; padding: 2px; border: 1px solid #333; }
.qty-btn { background: transparent; color: #fff; border: none; width: 28px; height: 28px; font-size: 1.1rem; font-weight: 600; cursor: pointer; }
.qty-num { font-weight: 700; min-width: 24px; text-align: center; }
.del-btn { background: rgba(255, 0, 51, 0.1); color: var(--hot-red); border: none; width: 32px; height: 32px; border-radius: 10px; cursor: pointer; display: flex; align-items: center; justify-content: center; }

.pedido-vacio { text-align: center; padding: 40px 20px; color: #888; }
.vacio-icono { font-size: 3.5rem; margin-bottom: 15px; opacity: 0.5; }
.pedido-vacio p { font-size: 1.2rem; font-weight: 700; color: #fff; margin: 0 0 8px 0; }

.modal-footer { border-top: 1px solid #2d2d2d; padding-top: 15px; }
.order-total { display: flex; justify-content: space-between; align-items: center; margin-bottom: 18px; }
.total-label { font-size: 1.1rem; color: #aaa; font-weight: 600; }
.total-monto { font-size: 1.8rem; font-weight: 900; color: var(--neon-yellow); }

.btn-group { display: flex; flex-direction: column; gap: 12px; }
.btn-enviar { background: var(--hot-red); color: white; padding: 16px; border-radius: 14px; width: 100%; border: none; font-weight: 700; font-size: 1.05rem; cursor: pointer; }
.btn-limpiar { background: transparent; color: #888888; border: none; font-weight: 600; cursor: pointer; text-decoration: underline; align-self: center; }

/* ESTILOS DE LA FACTURA */
.factura-container { font-family: 'Poppins', sans-serif; color: #111111; background: #ffffff; padding: 30px; width: 100%; box-sizing: border-box; }
.factura-header { text-align: center; border-bottom: 3px solid #111111; padding-bottom: 12px; margin-bottom: 20px; }
.factura-header h2 { font-weight: 900; font-size: 2rem; color: #ff0033; margin: 0; }
.factura-subtitle { font-size: 0.9rem; color: #555555; margin: 5px 0 0 0; font-weight: 600; text-transform: uppercase; }
.factura-meta { background: #f9f9f9; padding: 14px; border-radius: 10px; margin-bottom: 25px; border: 1px solid #eaeaea; display: flex; flex-direction: column; gap: 6px; }
.meta-item { font-size: 0.95rem; color: #333; display: flex; justify-content: space-between; }
.badge-status { background: #e6f9ed; color: #1ea94b; padding: 2px 8px; border-radius: 4px; font-weight: 700; }
.factura-tabla { width: 100%; border-collapse: collapse; margin-bottom: 25px; }
.factura-tabla th { background: #111111; color: #ffffff; padding: 10px 8px; font-weight: 700; }
.factura-tabla td { padding: 10px 8px; border-bottom: 1px solid #eaeaea; color: #111111; }
.factura-total-block { background: #f1f1f1; padding: 15px; border-radius: 8px; margin-bottom: 20px; }
.total-row { display: flex; justify-content: space-between; align-items: center; }
.total-block-label { font-weight: 700; font-size: 1rem; }
.total-block-monto { font-size: 1.4rem; color: #ff0033; }
.factura-footer { text-align: center; font-size: 0.85rem; color: #666666; border-top: 1px dashed #cccccc; padding-top: 15px; }
.gracias-msg { font-weight: 700; color: #111111; margin-bottom: 5px; }
</style>
