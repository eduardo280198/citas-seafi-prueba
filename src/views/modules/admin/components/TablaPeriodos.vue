<script setup>
import {onMounted, reactive, ref, watch} from "vue";
import useVuelidate from "@vuelidate/core/dist/index.esm";
import {helpers, required} from "@vuelidate/validators";
import flatPickr from "vue-flatpickr-component";
import Pickr from "@simonwep/pickr";
import "flatpickr/dist/flatpickr.css";
import "@simonwep/pickr/dist/themes/classic.min.css"; // 'classic' theme
import ModalComponent from "@/components/ModalComponent";
import {PeriodosService} from "@/api/connection/providers/Periodos.service";
import {UtilsDate} from "@/helpers/Dates/UtilsDate";
import NotificacionError from "@/helpers/notifications/NotificacionError";
import {NotificacionesModal} from "@/helpers/notifications/notificacionesModal";
import {NotificacionesRecepcion} from "@/helpers/notifications/NotificacionesRecepcion";
import {NotificacionesCitas} from "@/helpers/notifications/NotificacionCitas";
import PantallaCarga from "@/helpers/notifications/PantallaCarga";

const diasInhabiles = ref([])
const cargandoDatos = ref(false)
const respuestaBack = ref(null)
const paginaActual = ref(1)
const registrosPorPagina = ref(5)
const totalPaginas = ref(0)
const totalRegistros = ref(0)
const primerIndice = ref(0)
const ultimoIndice = ref(0)
const paginacion = ref()
const modalPeriodo = ref()
const datenow = new Date().toLocaleDateString();
const isNew = ref(true)

const cargarDias = async () => {
  try{
    cargandoDatos.value = true;
    if(state.searchQuery.length > 0) 
      paginaActual.value = 1;
      respuestaBack.value = await PeriodosService.ObtenerListaPeriodos(state.searchQuery, paginaActual.value, registrosPorPagina.value, 3, 'asc');
      paginaActual.value = respuestaBack.value.paginaActual;
      registrosPorPagina.value = respuestaBack.value.registrosPorPagina;
      totalPaginas.value = respuestaBack.value.totalPaginas;
      totalRegistros.value = respuestaBack.value.totalRegistros;
      diasInhabiles.value = respuestaBack.value.registros;
      primerIndice.value = respuestaBack.value.primerIndice;
      ultimoIndice.value = respuestaBack.value.ultimoIndice;
  } catch (e) {
    throw e;
  }finally {
    cargandoDatos.value = false;
  }
}

const cambiarPagina = (num) => {
  paginaActual.value = num;
  cargarDias();
}
watch(paginaActual, (newPage) => {
  cambiarPagina(newPage);
})

const agregarPeriodo = async (idPeriodo) => {
  await modalPeriodo.value.hide();

  if (idPeriodo !== ''){

    const resp = await NotificacionesRecepcion.ConfirmacionEditar(state.fechaInicio, 'Modificar');

    if(resp.isConfirmed){

      try{
        PantallaCarga.mostrar();
        const resp = await PeriodosService.modificarPeriodo(idPeriodo, state.nombre, UtilsDate.toYearMonthDay(state.fechaInicio), UtilsDate.toYearMonthDay(state.fechaFinal), state.activo);

        if(resp) {
          await NotificacionesModal.PantallaExito('Evento Editado con Éxito');
        }
        await limpiarModal();

      }catch (error){

        await NotificacionError.Agendar(error);

      }finally {
        PantallaCarga.ocultar();
        await cargarDias();
      }
    }
  } else{ // Evento nuevo

    try{
      const date = await NotificacionesRecepcion.ValidarDiaInhabil(state.fechaInicio, 'Agregar');

      if (date.isConfirmed) {
        try {
          PantallaCarga.mostrar();
          const resp = await PeriodosService.agregarPeriodo(idPeriodo, state.nombre, UtilsDate.toYearMonthDay(state.fechaInicio), UtilsDate.toYearMonthDay(state.fechaFinal), state.activo);
          await modalPeriodo.value.hide();

          if(resp){
            await NotificacionesModal.PantallaExito('Evento Agregado con Éxito');
          }

          await limpiarModal();

        }catch(error) {
          await NotificacionError.Agendar(error);
        }finally {
          PantallaCarga.ocultar();
          await cargarDias();
        }
      }
    }catch (e){

      const resp = await NotificacionesCitas.Continuar(e);

      if(resp.isConfirmed){
        try{
          PantallaCarga.mostrar();
          const resp = await PeriodosService.agregarPeriodo(idPeriodo, state.nombre, UtilsDate.toYearMonthDay(state.fechaInicio), UtilsDate.toYearMonthDay(state.fechaFinal), state.activo );
          await modalPeriodo.value.hide();

          if (resp) {
            await NotificacionesModal.PantallaExito('Evento Agregado con Éxito');

          }
          await limpiarModal();

        }catch (error){
          await NotificacionError.Agendar(error);
        }finally {
          PantallaCarga.ocultar();
          await cargarDias();
        }
      }
    }
  }
}

const eliminarPeriodo = async(idPeriodo, nombrePeriodo) => {
  const confirmado = await NotificacionesRecepcion.ConfirmacionEliminar(nombrePeriodo, 'Se eliminará');
  if(confirmado.isConfirmed) {

    try{
      const resp = await PeriodosService.eliminarPeriodo(idPeriodo);
      if(resp) {
        await NotificacionesModal.PantallaExito('Periodo Eliminado con Éxito');
      }
    } catch (e) {
      await NotificacionError.Agendar(error);

    }finally {

      await cargarDias();
    }
  }
}

const state = reactive( {
  searchQuery: '',
  idPeriodo: '',
  nombre: '',
  activo: true,
  fechaInicio: datenow,
  fechaFinal: datenow,
});

const rules = {
  searchQuery: {},
  nombre: {
    required: helpers.withMessage("Este campo es obligatorio", required)
  },
  fechaInicio: {
    required: helpers.withMessage("Este campo es obligatorio", required),
  },
  fechaFinal: {
    required: helpers.withMessage("Este campo es obligatorio", required),
  },
  activo: {}
}

const v$ = useVuelidate(rules, state);

const humanfriendlyConfig = {
  defaultDate: 'today',
  dateFormat: 'd/m/Y',
  // minDate: new Date(new Date().getFullYear() - 1, 12, 1),
  // maxDate: new Date(new Date().getFullYear() + 1, 12, 1)
}

const mostrarModal = async (idPeriodo, index) => {
  let inputInicio = document.querySelector('#inputfechaInicio');
  let inputFin    = document.querySelector('#inputfechaFin');
  await limpiarModal()

  if(!idPeriodo) {
    isNew.value = true;
    inputInicio.disabled = false;
    inputFin.disabled = false;
  }else{

    isNew.value = false;
    inputInicio.disabled = true;
    inputFin.disabled = true;

    //const dato = await diasInhabiles.value[index];
    const dato = idPeriodo;

    state.idPeriodo = dato.idPeriodo;
    state.nombre = dato.nombrePeriodo;

    state.fechaInicio = UtilsDate.toDayMonthYear(dato.fechaInicio);
    state.fechaFinal = UtilsDate.toDayMonthYear(dato.fechaFin);

    state.activo = dato.activo;

  }
  await modalPeriodo.value.show();
}

const limpiarModal = async() => {
  v$.value.$reset();
  state.idPeriodo = '';
  state.nombre = '';
  state.activo = true;
  state.fechaInicio = datenow;
  state.fechaFinal = datenow;
}

onMounted(() => {
  cargarDias();
});

</script>

<template>
  <div class="card">

    <!-- Boton de Nuevo Evento -->
    <div class="card-header align-items-center d-flex">
      <h3 class="mb-0 flex-grow-1">Periodos</h3>
      <div class="flex-shrink-0">
        <div class="form-check form-switch form-switch-right form-switch-md">
          <b-button @click="mostrarModal()" variant="pantone-green">
            <i class="bx bx-plus-circle fs-2 align-middle"></i>Nuevo Periodo
          </b-button>
        </div>
      </div>
    </div> <!-- Fin del Boton -->

    <!-- Barra de Busqueda -->
    <div class="card-body border border-dashed border-end-0 border-start-0">
      <div class="row g-3 d-flex justify-content-end">
        <div class="col-md-6 col-sm-12 col-lg-4">
          <div class="search-box">
            <input v-model="v$.searchQuery.$model" type="text" class="form-control search bg-light border-light"
              placeholder="Buscar Periodo" @keyup.stop="cargarDias()"
            />
            <i class="ri-search-line search-icon"></i>
          </div>
        </div>
      </div>
    </div> <!-- Final de Barra de Busqueda-->

    <div class="card-body">
      <div class="flex-grow-1 overflow-hidden">
        <div class="table-responsive">
          <table class="table align-middle table-nowrap mb-0 table-hover mb-4">
            <thead class="text-muted">
              <tr>
                <th class="sort">ID</th>
                <th class="sort">Nombre</th>
                <th class="sort text-center">Fecha Inicio</th>
                <th class="sort text-center">Fecha Final</th>
                <th class="sort text-center">Activo</th>
                <th class="sort text-center">Acciones</th>
              </tr>
            </thead>


            <!-- Buscar el Error por aqui el MARTES! -->
           <tbody>
            <tr v-for="(periodo, index) in diasInhabiles">
              <th scope="row">{{ index + 1 }}</th>

              <td class="fw-semibold">{{ periodo.nombrePeriodo }}</td>
              <td class="text-center">{{ UtilsDate.toDayMonthYear(periodo.fechaInicio) }}</td>
              <td class="text-center">{{ UtilsDate.toDayMonthYear(periodo.fechaFin) }}</td>

              <td class="text-muted text-center">
                <i :class="periodo.activo === true ? 'mdi mdi-checkbox-marked fs-1 text-seafi-aqua' : 'mdi mdi-close-box fs-1'"></i>
              </td>

              <td>

                <div class="text-center">
                  <button class="btn btn-light btn-sm dropdown" type="button"
                          data-bs-toggle="dropdown" aria-expanded="false">
                    <i class="mdi mdi-dots-horizontal"></i>      
                  </button>

                  <ul class="dropdown-menu">

                    <li>
                      <a class="dropdown-item" style="cursor: pointer" @click="mostrarModal(periodo, index)">
                        <i class="mdi mdi-calendar-edit mdi-18px align-center me-2 text-seafi-aqua"></i>
                        Editar
                      </a>
                    </li>

                    <li>
                      <a class="dropdown-item" style="cursor: pointer" @click="eliminarPeriodo(periodo.idPeriodo, periodo.nombrePeriodo)">
                        <i class="mdi mdi-calendar-remove mdi-18px align-center me-2 text-seafi-red"></i>
                        Eliminar
                      </a>
                    </li>
                  </ul>

                </div>

              </td>

            </tr>
           </tbody> 
           
          </table>
        </div>
      </div>

      <div class="text-center" v-if="cargandoDatos">
        <div class="spinner">
          <div class="bounce1"></div>
          <div class="bounce2"></div>
          <div class="bounce3"></div>
        </div>
        <br>
      </div>

      <div v-else>
        <div class="noresult" v-if="diasInhabiles.length === 0">
          <div class="text-center">
            <h5 class="mt-2">No hay registros</h5>
            <p class="text-muted mb-0">
              No se encontraron resultados para mostrar
            </p>
          </div>
        </div>

        <div class="row g-0 text-center text-sm-start align-items-center mb-4" v-else>
          <div class="col-sm-6">
            
            <p class="mb-sm-0 text-muted">
              Mostrando del <span class="fw-semibold">{{primerIndice}}</span> al
              <span class="fw-semibold">{{ultimoIndice}}</span> de
              <span class="fw-semibold text-decoration-underline">{{totalRegistros}}</span>
              registros
            </p>
          </div>

          <div class="col-sm-6">
            <ul class="pagination pagination-separated justify-content-center justify-content-sm-end mb-sm-0">
              <b-pagination
                ref="paginacion"
                v-model="paginaActual"
                :total-rows="totalRegistros"
                :per-page="registrosPorPagina"
                first-number
                last-number
              ></b-pagination>
            </ul>
          </div>
        </div>
      </div>

    </div>

    <ModalComponent ref="modalPeriodo">

      <template #header>
        <h5 class="modal-title"><span>{{ isNew === true ? 'Nuevo' : 'Modificar' }} periodo</span></h5>
        <button class="btn-close" type="button" data-bs-dismiss="modal" aria-label="Close"></button>
      </template>

      <template #body>
        <div class="text-center">
          <i class="mdi text-seafi-aqua" :class="isNew ? 'mdi-calendar-xpand-horizontal' : 'mdi-calendar-edit'" style="font-size:100px"></i>
        </div>

        <div class="mb-3">
          <label class="form-label">Nombre del Período
            <span class="text-danger">*</span>
          </label>

          <input type="text" class="form-control" v-model.trim="v$.nombre.$model"
                 
                 :class="{'is-invalid': v$.nombre.$error,}" id="nombreSistema"
                 placeholder="Ingresa un nombre para el período">
          
          <div v-for="(item, index) in v$.nombre.$errors" :key="index"
               class="invalid-feedback">
            <span class="text-danger" v-if="item.$message">{{ item.$message }}</span>
          </div>
        </div>

        <div class="mb-3">
          <label for="zipcodeInput" class="form-label">Fecha Inicio <span
            class="text-danger">*</span>
          </label>
          
          <flatPickr
             v-model="v$.fechaInicio.$model"
             :config="humanfriendlyConfig"
             id="inputfechaInicio"
             class="form-control flatpickr-input"
             :class="{'is-invalid': v$.fechaInicio.$error}, !isNew ? 'text-muted' : ''"
             :style="!isNew ? 'cursor:not-allowed' : ''"
          ></flatPickr>

          <div v-for="(item, index) in v$.fechaInicio.$errors" :key="index"
               class="invalid-feedback">
            <span class="text-danger" v-if="item.$message">{{ item.$message }}</span>
          </div>
        </div>

        <div class="mb-3">
          <label for="zipcodeInput" class="form-label">Fecha Fin <span
             class="text-danger">*</span>
          </label>

          <flatPickr
               v-model="v$.fechaFinal.$model"
               :config="humanfriendlyConfig"
               id="inputfechaFin"
               class="form-control flatpickr-input"
               :class="{'is-invalid':v$.fechaFinal.$error}, !isNew ? 'text-muted' : '' "
               :style="!isNew ? 'cursor: not-allowed' : ''"
          ></flatPickr>

          <div v-for="(item, index) in v$.fechaFinal.$errors" :key="index"
               class="invalid-feedback">
            <span class="text-danger" v-if="item.$message">{{ item.$message }}</span>
          </div>
        </div>

        <div class="mb-3">

          <div class="form-switch form-switch-lg form-check-seafi-aqua" dir="ltr">
            <input type="checkbox" class="form-check-input" id="estaActivo"
                   v-model="v$.activo.$model" :class="{'is-invalid':v$.activo.$error,}">
            <label for="estaActivo" class="form-check-label">Activo <span
                 class="text-danger">*</span>
            </label>
            
            <div v-for="(item, index) in v$.activo.$errors" :key="index"
                 class="invalid-feedback">
              <span class="text-danger" v-if="item.$message">{{ item.$message }}</span>    
            </div>

          </div>
        </div>

      </template>

      <template #footer>
        <div class="d-flex align-items-center col-12">
          <div class="col-6 px-2">
            <button class="btn btn-seafi-aqua w-100" ref="enviando"
               type="button"
               :disabled="v$.$invalid"
               @click="agregarPeriodo(state.idPeriodo)">Aceptar
            </button>

          </div>

          <div class="col-6 px-2">
            <a class="btn btn-soft-seafi-gray w-100" data-bs-dismiss="modal">Cancelar</a>
          </div>
        </div>
      </template>

    </ModalComponent>
    
  </div>
</template>