# Software-FJ-Gestion
from abc import ABC, abstractmethod
from datetime import datetime
import os

# ==============================================
# 1. DEFINICIÓN DE EXCEPCIONES
# ==============================================
class ErrorSistema(Exception):
    """Excepción base para todos los errores del sistema"""
    pass

class ErrorClienteInvalido(ErrorSistema):
    """Error: Datos del cliente incorrectos o incompletos"""
    pass

class ErrorServicioInvalido(ErrorSistema):
    """Error: Datos del servicio incorrectos o no existe"""
    pass

class ErrorReservaInvalida(ErrorSistema):
    """Error: Datos de la reserva inválidos o operación no permitida"""
    pass

class ErrorDuracionInvalida(ErrorSistema):
    """Error: Duración ingresada es negativa o cero"""
    pass

# ==============================================
# 2. GESTOR DE LOGS (ARCHIVO DE REGISTRO)
# ==============================================
class GestorLog:
    """Clase encargada de registrar todos los eventos y errores en archivo"""
    ARCHIVO_LOG = "registro_logs.txt"

    @classmethod
    def registrar(cls, tipo_evento, mensaje):
        """Guarda un registro con fecha y hora en el archivo"""
        try:
            fecha_hora = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            linea = f"[{fecha_hora}] | {tipo_evento.upper()} | {mensaje}\n"
            
            # Abrir archivo en modo agregar, si no existe lo crea
            with open(cls.ARCHIVO_LOG, "a", encoding="utf-8") as archivo:
                archivo.write(linea)
        
        except Exception as e:
            # Si falla el log, no detener el sistema
            print(f"⚠️ No se pudo guardar el registro: {str(e)}")

# ==============================================
# 3. CLASES ABSTRACTAS Y ESTRUCTURA
# ==============================================

# ---- CLASE ABSTRACTA BASE ----
class EntidadBase(ABC):
    """Clase abstracta que representa entidades generales del sistema"""
    def __init__(self, identificador, nombre):
        self._identificador = identificador  # Atributo protegido (encapsulamiento)
        self._nombre = nombre

    @abstractmethod
    def obtener_informacion(self):
        """Método abstracto: todas las clases hijas deben implementarlo"""
        pass

    # Métodos de acceso controlado
    def obtener_id(self):
        return self._identificador

    def obtener_nombre(self):
        return self._nombre

# ---- CLASE CLIENTE ----
class Cliente(EntidadBase):
    """Clase que representa un cliente, con validaciones estrictas y encapsulamiento"""
    def __init__(self, identificador, nombre, correo, telefono):
        # Validaciones antes de crear el objeto
        self._validar_datos(identificador, nombre, correo, telefono)
        super().__init__(identificador, nombre)
        self.__correo = correo       # Atributo privado
        self.__telefono = telefono    # Atributo privado

    def _validar_datos(self, identificador, nombre, correo, telefono):
        """Validaciones internas de datos del cliente"""
        if not identificador or not str(identificador).strip():
            raise ErrorClienteInvalido("El número de identificación no puede estar vacío")
        if not nombre or len(nombre.strip()) < 3:
            raise ErrorClienteInvalido("El nombre debe tener al menos 3 caracteres")
        if "@" not in correo or "." not in correo:
            raise ErrorClienteInvalido("El correo electrónico ingresado no es válido")
        if not telefono or not telefono.isdigit() or len(telefono) < 7:
            raise ErrorClienteInvalido("El teléfono debe ser numérico y tener al menos 7 dígitos")

    def obtener_informacion(self):
        """Implementación método abstracto: datos completos del cliente"""
        return f"ID: {self._identificador} | Nombre: {self._nombre} | Correo: {self.__correo} | Teléfono: {self.__telefono}"

    # Métodos sobrescritos para acceso seguro
    def obtener_correo(self):
        return self.__correo

    def obtener_telefono(self):
        return self.__telefono

# ---- CLASE ABSTRACTA SERVICIO + HIJAS ----
class Servicio(EntidadBase):
    """Clase abstracta para servicios, base de todos los tipos de servicio"""
    def __init__(self, identificador, nombre, precio_base):
        super().__init__(identificador, nombre)
        self._precio_base = precio_base
        self._validar_precio()

    def _validar_precio(self):
        if self._precio_base < 0:
            raise ErrorServicioInvalido("El precio base no puede ser negativo")

    @abstractmethod
    def calcular_costo(self, duracion, **kwargs):
        """Método abstracto: cálculo de costo, con polimorfismo"""
        pass

    @abstractmethod
    def describir_servicio(self):
        """Método abstracto: descripción del servicio"""
        pass

# SERVICIO 1: ALQUILER DE SALAS
class ServicioSala(Servicio):
    def __init__(self, identificador, nombre, precio_base, capacidad):
        super().__init__(identificador, nombre, precio_base)
        self.__capacidad = capacidad

    # Método SOBRECARGA: diferentes formas de calcular costo
    def calcular_costo(self, duracion, impuesto=0.16, descuento=0):
        """Cálculo con impuesto y descuento"""
        if duracion <= 0:
            raise ErrorDuracionInvalida("La duración debe ser mayor a cero")
        costo_bruto = self._precio_base * duracion
        costo_con_impuesto = costo_bruto * (1 + impuesto)
        costo_final = costo_con_impuesto * (1 - descuento)
        return round(costo_final, 2)

    def calcular_costo_simple(self, duracion):
        """Versión simplificada sin impuesto"""
        return round(self._precio_base * duracion, 2)

    def describir_servicio(self):
        return f"Servicio: {self._nombre} | Capacidad: {self.__capacidad} personas | Precio base: ${self._precio_base}"

    def obtener_informacion(self):
        return self.describir_servicio()

# SERVICIO 2: ALQUILER DE EQUIPOS
class ServicioEquipo(Servicio):
    def __init__(self, identificador, nombre, precio_base, tipo_equipo):
        super().__init__(identificador, nombre, precio_base)
        self.__tipo_equipo = tipo_equipo

    def calcular_costo(self, duracion, impuesto=0.16, seguro=True):
        """Cálculo con opción de seguro"""
        if duracion <= 0:
            raise ErrorDuracionInvalida("La duración debe ser mayor a cero")
        costo_bruto = self._precio_base * duracion
        if seguro:
            costo_bruto += 50  # Cargo por seguro
        costo_final = costo_bruto * (1 + impuesto)
        return round(costo_final, 2)

    def describir_servicio(self):
        return f"Servicio: {self._nombre} | Tipo: {self.__tipo_equipo} | Precio base: ${self._precio_base}"

    def obtener_informacion(self):
        return self.describir_servicio()

# SERVICIO 3: ASESORÍA ESPECIALIZADA
class ServicioAsesoria(Servicio):
    def __init__(self, identificador, nombre, precio_base, especialidad):
        super().__init__(identificador, nombre, precio_base)
        self.__especialidad = especialidad

    def calcular_costo(self, duracion, impuesto=0.16, recargo_urgencia=0):
        """Cálculo con recargo opcional por urgencia"""
        if duracion <= 0:
            raise ErrorDuracionInvalida("La duración debe ser mayor a cero")
        costo_bruto = self._precio_base * duracion
        costo_bruto += costo_bruto * recargo_urgencia
        costo_final = costo_bruto * (1 + impuesto)
        return round(costo_final, 2)

    def describir_servicio(self):
        return f"Servicio: {self._nombre} | Especialidad: {self.__especialidad} | Precio base: ${self._precio_base}"

    def obtener_informacion(self):
        return self.describir_servicio()

# ---- CLASE RESERVA ----
class Reserva(EntidadBase):
    """Clase que integra cliente, servicio, duración y estado; maneja todo el ciclo"""
    ESTADOS = ["PENDIENTE", "CONFIRMADA", "CANCELADA", "FINALIZADA"]

    def __init__(self, identificador, cliente, servicio, duracion):
        super().__init__(identificador, f"Reserva-{identificador}")
        self.__cliente = cliente
        self.__servicio = servicio
        self.__duracion = duracion
        self.__estado = "PENDIENTE"
        self.__costo_total = 0
        self._validar_reserva()

    def _validar_reserva(self):
        """Validaciones internas al crear reserva"""
        if not isinstance(self.__cliente, Cliente):
            raise ErrorReservaInvalida("El cliente asignado no es válido")
        if not isinstance(self.__servicio, Servicio):
            raise ErrorReservaInvalida("El servicio asignado no es válido")
        if self.__duracion <= 0:
            raise ErrorDuracionInvalida("La duración de la reserva debe ser mayor a cero")

    # Métodos principales
    def confirmar_reserva(self, **parametros_calculo):
        """Confirma la reserva y calcula el costo"""
        try:
            if self.__estado != "PENDIENTE":
                raise ErrorReservaInvalida("Solo se pueden confirmar reservas pendientes")
            
            self.__costo_total = self.__servicio.calcular_costo(self.__duracion, **parametros_calculo)
            self.__estado = "CONFIRMADA"
            GestorLog.registrar("ÉXITO", f"Reserva {self._identificador} CONFIRMADA. Costo: ${self.__costo_total}")
            return True

        except ErrorSistema as e:
            GestorLog.registrar("ERROR", f"Al confirmar reserva {self._identificador}: {str(e)}")
            raise

    def cancelar_reserva(self):
        """Cancela la reserva si está permitido"""
        try:
            if self.__estado == "FINALIZADA":
                raise ErrorReservaInvalida("No se puede cancelar una reserva ya finalizada")
            self.__estado = "CANCELADA"
            GestorLog.registrar("AVISO", f"Reserva {self._identificador} CANCELADA")
            return True

        except ErrorSistema as e:
            GestorLog.registrar("ERROR", f"Al cancelar reserva {self._identificador}: {str(e)}")
            raise

    def obtener_informacion(self):
        """Datos completos de la reserva"""
        return (
            f"=== RESERVA #{self._identificador} ===\n"
            f"Estado: {self.__estado}\n"
            f"Duración: {self.__duracion} horas\n"
            f"Costo Total: ${self.__costo_total}\n"
            f"--- Cliente ---\n{self.__cliente.obtener_informacion()}\n"
            f"--- Servicio ---\n{self.__servicio.obtener_informacion()}"
        )

    # Métodos sobrescritos
    def obtener_datos_resumidos(self):
        return f"Reserva #{self._identificador} | Estado: {self.__estado} | Cliente: {self.__cliente.obtener_nombre()}"

# ==============================================
# 4. SISTEMA PRINCIPAL (GESTIÓN Y MENÚ)
# ==============================================
class SistemaGestion:
    """Clase principal: administra todas las listas, menú y operaciones"""
    def __init__(self):
        # Almacenamiento SOLO en listas (sin base de datos)
        self.clientes = []
        self.servicios = []
        self.reservas = []
        self.contador_ids = {"cliente": 1000, "servicio": 2000, "reserva": 3000}

        # Limpiar log anterior (opcional)
        if os.path.exists(GestorLog.ARCHIVO_LOG):
            pass  # No borrar, seguir agregando registros

    # ---------- MÉTODOS DE BÚSQUEDA ----------
    def buscar_cliente_por_id(self, id_buscar):
        for c in self.clientes:
            if c.obtener_id() == id_buscar:
                return c
        return None

    def buscar_servicio_por_id(self, id_buscar):
        for s in self.servicios:
            if s.obtener_id() == id_buscar:
                return s
        return None

    def buscar_reserva_por_id(self, id_buscar):
        for r in self.reservas:
            if r.obtener_id() == id_buscar:
                return r
        return None

    # ---------- OPERACIONES DE NEGOCIO ----------
    def registrar_cliente(self, nombre, correo, telefono):
        """Registra cliente con manejo completo de errores"""
        try:
            nuevo_id = self.contador_ids["cliente"]
            cliente_nuevo = Cliente(nuevo_id, nombre, correo, telefono)
            self.clientes.append(cliente_nuevo)
            self.contador_ids["cliente"] += 1
            GestorLog.registrar("ÉXITO", f"Cliente registrado ID: {nuevo_id} - Nombre: {nombre}")
            print(f"✅ Cliente registrado correctamente. ID asignado: {nuevo_id}")
            return cliente_nuevo

        except ErrorClienteInvalido as e:
            GestorLog.registrar("ERROR", f"Registro cliente fallido: {str(e)}")
            print(f"❌ Error al registrar: {str(e)}")
        except Exception as e:
            GestorLog.registrar("ERROR", f"Error inesperado cliente: {str(e)}")
            print(f"⚠️ Error interno: {str(e)}")

    def crear_servicio(self, tipo, nombre, precio_base, **datos_extra):
        """Crea cualquier tipo de servicio, usando polimorfismo"""
        try:
            nuevo_id = self.contador_ids["servicio"]
            servicio = None

            if tipo.upper() == "SALA":
                servicio = ServicioSala(nuevo_id, nombre, precio_base, datos_extra.get("capacidad", 10))
            elif tipo.upper() == "EQUIPO":
                servicio = ServicioEquipo(nuevo_id, nombre, precio_base, datos_extra.get("tipo", "Genérico"))
            elif tipo.upper() == "ASESORIA":
                servicio = ServicioAsesoria(nuevo_id, nombre, precio_base, datos_extra.get("especialidad", "General"))
            else:
                raise ErrorServicioInvalido("Tipo de servicio no reconocido")

            self.servicios.append(servicio)
            self.contador_ids["servicio"] += 1
            GestorLog.registrar("ÉXITO", f"Servicio creado ID: {nuevo_id} - Tipo: {tipo}")
            print(f"✅ Servicio creado correctamente. ID asignado: {nuevo_id}")
            return servicio

        except ErrorSistema as e:
            GestorLog.registrar("ERROR", f"Creación servicio fallida: {str(e)}")
            print(f"❌ Error: {str(e)}")

    def crear_reserva(self, id_cliente, id_servicio, duracion):
        """Crea reserva integrando todo"""
        try:
            cliente = self.buscar_cliente_por_id(id_cliente)
            servicio = self.buscar_servicio_por_id(id_servicio)

            if not cliente:
                raise ErrorReservaInvalida("El cliente ingresado no está registrado en el sistema")
            if not servicio:
                raise ErrorReservaInvalida("El servicio ingresado no existe en el sistema")
            if duracion <= 0:
                raise ErrorDuracionInvalida("La duración debe ser mayor a cero horas")

            nuevo_id = self.contador_ids["reserva"]
            reserva_nueva = Reserva(nuevo_id, cliente, servicio, duracion)
            self.reservas.append(reserva_nueva)
            self.contador_ids["reserva"] += 1
            GestorLog.registrar("ÉXITO", f"Reserva creada ID: {nuevo_id} - Cliente: {cliente.obtener_nombre()}")
            print(f"✅ Reserva creada correctamente. ID asignado: {nuevo_id}")
            return reserva_nueva

        except ErrorSistema as e:
            GestorLog.registrar("ERROR", f"Creación reserva fallida: {str(e)}")
            print(f"❌ Error: {str(e)}")
        except Exception as e:
            GestorLog.registrar("ERROR", f"Error inesperado reserva: {str(e)}")
            print(f"⚠️ Error interno: {str(e)}")

    def confirmar_reserva(self, id_reserva, **parametros):
        """Confirma una reserva existente"""
        try:
            reserva = self.buscar_reserva_por_id(id_reserva)
            if not reserva:
                raise ErrorReservaInvalida("Reserva no encontrada")
            
            reserva.confirmar_reserva(**parametros)
            print("✅ Reserva confirmada y costo calculado correctamente")
            return True

        except ErrorSistema as e:
            print(f"❌ No se pudo confirmar: {str(e)}")
        except Exception as e:
            print(f"⚠️ Error interno: {str(e)}")

    def cancelar_reserva(self, id_reserva):
        """Cancela una reserva existente"""
        try:
            reserva = self.buscar_reserva_por_id(id_reserva)
            if not reserva:
                raise ErrorReservaInvalida("Reserva no encontrada")
            
            reserva.cancelar_reserva()
            print("✅ Reserva cancelada correctamente")
            return True

        except ErrorSistema as e:
            print(f"❌ No se pudo cancelar: {str(e)}")

    # ---------- MÉTODOS DE EDICIÓN (UPDATE) ----------
    def editar_cliente(self, id_cliente, nombre, correo, telefono):
        """Edita datos de un cliente existente"""
        try:
            cliente = self.buscar_cliente_por_id(id_cliente)
            if not cliente:
                raise ErrorClienteInvalido("Cliente no encontrado")
            
            # Aplicar validaciones
            cliente._validar_datos(id_cliente, nombre, correo, telefono)
            
            # Actualizar atributos
            cliente._nombre = nombre
            cliente._Cliente__correo = correo  # Name mangling para atributo privado
            cliente._Cliente__telefono = telefono
            
            GestorLog.registrar("ÉXITO", f"Cliente {id_cliente} editado: {nombre}")
            print(f"✅ Cliente {id_cliente} actualizado correctamente")
            return True

        except ErrorClienteInvalido as e:
            GestorLog.registrar("ERROR", f"Edición cliente {id_cliente} fallida: {str(e)}")
            print(f"❌ Error: {str(e)}")
        except Exception as e:
            GestorLog.registrar("ERROR", f"Error editar cliente: {str(e)}")
            print(f"⚠️ Error interno: {str(e)}")

    def editar_servicio(self, id_servicio, nombre, precio_base):
        """Edita datos de un servicio existente (nombre y precio)"""
        try:
            servicio = self.buscar_servicio_por_id(id_servicio)
            if not servicio:
                raise ErrorServicioInvalido("Servicio no encontrado")
            
            # Validar precio
            if precio_base < 0:
                raise ErrorServicioInvalido("El precio base no puede ser negativo")
            
            # Actualizar
            servicio._nombre = nombre
            servicio._precio_base = precio_base
            
            GestorLog.registrar("ÉXITO", f"Servicio {id_servicio} editado: {nombre}")
            print(f"✅ Servicio {id_servicio} actualizado correctamente")
            return True

        except ErrorServicioInvalido as e:
            GestorLog.registrar("ERROR", f"Edición servicio {id_servicio} fallida: {str(e)}")
            print(f"❌ Error: {str(e)}")
        except Exception as e:
            GestorLog.registrar("ERROR", f"Error editar servicio: {str(e)}")
            print(f"⚠️ Error interno: {str(e)}")

    def editar_reserva(self, id_reserva, duracion):
        """Edita la duración de una reserva (solo si está PENDIENTE o CONFIRMADA)"""
        try:
            reserva = self.buscar_reserva_por_id(id_reserva)
            if not reserva:
                raise ErrorReservaInvalida("Reserva no encontrada")
            
            # Validar duración
            if duracion <= 0:
                raise ErrorDuracionInvalida("La duración debe ser mayor a cero")
            
            # Obtener estado actual (acceso limitado)
            estado_actual = reserva._Reserva__estado
            if estado_actual == "CANCELADA" or estado_actual == "FINALIZADA":
                raise ErrorReservaInvalida(f"No se puede editar una reserva {estado_actual}")
            
            # Actualizar duración
            reserva._Reserva__duracion = duracion
            reserva._Reserva__costo_total = 0  # Resetear costo para que se recalcule
            
            GestorLog.registrar("ÉXITO", f"Reserva {id_reserva} editada: duración {duracion} horas")
            print(f"✅ Reserva {id_reserva} actualizada correctamente")
            return True

        except ErrorReservaInvalida as e:
            GestorLog.registrar("ERROR", f"Edición reserva {id_reserva} fallida: {str(e)}")
            print(f"❌ Error: {str(e)}")
        except ErrorDuracionInvalida as e:
            GestorLog.registrar("ERROR", f"Duración inválida en reserva {id_reserva}: {str(e)}")
            print(f"❌ Error: {str(e)}")
        except Exception as e:
            GestorLog.registrar("ERROR", f"Error editar reserva: {str(e)}")
            print(f"⚠️ Error interno: {str(e)}")

    # ---------- MÉTODOS DE ELIMINACIÓN (DELETE) ----------
    def eliminar_cliente(self, id_cliente):
        """Elimina un cliente (solo si no tiene reservas CONFIRMADAS o FINALIZADAS)"""
        try:
            cliente = self.buscar_cliente_por_id(id_cliente)
            if not cliente:
                raise ErrorClienteInvalido("Cliente no encontrado")
            
            # Validar que no tenga reservas activas
            reservas_cliente = [r for r in self.reservas if r._Reserva__cliente.obtener_id() == id_cliente]
            reservas_activas = [r for r in reservas_cliente if r._Reserva__estado in ["CONFIRMADA", "FINALIZADA"]]
            
            if reservas_activas:
                raise ErrorClienteInvalido(
                    f"No se puede eliminar: cliente tiene {len(reservas_activas)} reserva(s) activa(s)"
                )
            
            # Eliminar
            self.clientes.remove(cliente)
            GestorLog.registrar("ÉXITO", f"Cliente {id_cliente} eliminado: {cliente.obtener_nombre()}")
            print(f"✅ Cliente {id_cliente} eliminado correctamente")
            return True

        except ErrorClienteInvalido as e:
            GestorLog.registrar("AVISO", f"No se pudo eliminar cliente {id_cliente}: {str(e)}")
            print(f"❌ No se puede eliminar: {str(e)}")
        except Exception as e:
            GestorLog.registrar("ERROR", f"Error eliminar cliente: {str(e)}")
            print(f"⚠️ Error interno: {str(e)}")

    def eliminar_servicio(self, id_servicio):
        """Elimina un servicio (solo si no tiene reservas activas)"""
        try:
            servicio = self.buscar_servicio_por_id(id_servicio)
            if not servicio:
                raise ErrorServicioInvalido("Servicio no encontrado")
            
            # Validar que no tenga reservas asociadas activas
            reservas_servicio = [r for r in self.reservas if r._Reserva__servicio.obtener_id() == id_servicio]
            reservas_activas = [r for r in reservas_servicio if r._Reserva__estado in ["PENDIENTE", "CONFIRMADA"]]
            
            if reservas_activas:
                raise ErrorServicioInvalido(
                    f"No se puede eliminar: servicio tiene {len(reservas_activas)} reserva(s) activa(s)"
                )
            
            # Eliminar
            self.servicios.remove(servicio)
            GestorLog.registrar("ÉXITO", f"Servicio {id_servicio} eliminado: {servicio.obtener_nombre()}")
            print(f"✅ Servicio {id_servicio} eliminado correctamente")
            return True

        except ErrorServicioInvalido as e:
            GestorLog.registrar("AVISO", f"No se pudo eliminar servicio {id_servicio}: {str(e)}")
            print(f"❌ No se puede eliminar: {str(e)}")
        except Exception as e:
            GestorLog.registrar("ERROR", f"Error eliminar servicio: {str(e)}")
            print(f"⚠️ Error interno: {str(e)}")

    def eliminar_reserva(self, id_reserva):
        """Elimina una reserva (solo si está PENDIENTE)"""
        try:
            reserva = self.buscar_reserva_por_id(id_reserva)
            if not reserva:
                raise ErrorReservaInvalida("Reserva no encontrada")
            
            # Validar estado
            estado_actual = reserva._Reserva__estado
            if estado_actual != "PENDIENTE":
                raise ErrorReservaInvalida(
                    f"Solo se pueden eliminar reservas PENDIENTE (esta está {estado_actual})"
                )
            
            # Eliminar
            self.reservas.remove(reserva)
            GestorLog.registrar("ÉXITO", f"Reserva {id_reserva} eliminada")
            print(f"✅ Reserva {id_reserva} eliminada correctamente")
            return True

        except ErrorReservaInvalida as e:
            GestorLog.registrar("AVISO", f"No se pudo eliminar reserva {id_reserva}: {str(e)}")
            print(f"❌ No se puede eliminar: {str(e)}")
        except Exception as e:
            GestorLog.registrar("ERROR", f"Error eliminar reserva: {str(e)}")
            print(f"⚠️ Error interno: {str(e)}")

    # ---------- MÉTODOS PARA MOSTRAR DATOS ----------
    def mostrar_clientes(self):
        if not self.clientes:
            print("📋 No hay clientes registrados aún.")
            return
        print("\n=== LISTA DE CLIENTES ===")
        for c in self.clientes:
            print(f"- {c.obtener_informacion()}")

    def mostrar_servicios(self):
        if not self.servicios:
            print("📋 No hay servicios creados aún.")
            return
        print("\n=== LISTA DE SERVICIOS ===")
        for s in self.servicios:
            print(f"- {s.obtener_informacion()}")

    def mostrar_reservas(self):
        if not self.reservas:
            print("📋 No hay reservas registradas aún.")
            return
        print("\n=== LISTA DE TODAS LAS RESERVAS ===")
        for r in self.reservas:
            print(r.obtener_informacion())
            print("-" * 50)

    # ---------- MENÚ INTERACTIVO PRINCIPAL ----------
    def menu_principal(self):
        """Menú interactivo completo para que el usuario ingrese datos"""
        while True:
            print("\n" + "="*50)
            print("📚 SOFWARE FJ - MENÚ PRINCIPAL")
            print("="*50)
            print("\n--- CLIENTES ---")
            print("1. Registrar nuevo cliente")
            print("2. Ver lista de clientes")
            print("3. Editar cliente")
            print("4. Eliminar cliente")
            print("\n--- SERVICIOS ---")
            print("5. Crear nuevo servicio")
            print("6. Ver lista de servicios")
            print("7. Editar servicio")
            print("8. Eliminar servicio")
            print("\n--- RESERVAS ---")
            print("9. Crear nueva reserva")
            print("10. Ver todas las reservas")
            print("11. Editar reserva")
            print("12. Confirmar reserva")
            print("13. Cancelar reserva")
            print("14. Eliminar reserva")
            print("\n--- SISTEMA ---")
            print("15. Salir del sistema")
            print("="*50)

            try:
                opcion = int(input("🔹 Ingrese una opción (1-15): "))

                if opcion == 1:
                    print("\n--- REGISTRAR CLIENTE ---")
                    nombre = input("Nombre completo: ")
                    correo = input("Correo electrónico: ")
                    telefono = input("Teléfono: ")
                    self.registrar_cliente(nombre, correo, telefono)

                elif opcion == 2:
                    self.mostrar_clientes()

                elif opcion == 3:
                    print("\n--- EDITAR CLIENTE ---")
                    id_cliente = int(input("ID del cliente a editar: "))
                    nombre = input("Nuevo nombre completo: ")
                    correo = input("Nuevo correo electrónico: ")
                    telefono = input("Nuevo teléfono: ")
                    self.editar_cliente(id_cliente, nombre, correo, telefono)

                elif opcion == 4:
                    print("\n--- ELIMINAR CLIENTE ---")
                    id_cliente = int(input("ID del cliente a eliminar: "))
                    confirmacion = input(f"¿Confirma eliminar cliente {id_cliente}? (s/n): ")
                    if confirmacion.lower() == "s":
                        self.eliminar_cliente(id_cliente)
                    else:
                        print("❌ Operación cancelada")

                elif opcion == 5:
                    print("\n--- CREAR SERVICIO ---")
                    print("Tipos disponibles: SALA | EQUIPO | ASESORIA")
                    tipo = input("Ingrese tipo de servicio: ")
                    nombre = input("Nombre del servicio: ")
                    precio = float(input("Precio base ($): "))

                    datos_extra = {}
                    if tipo.upper() == "SALA":
                        datos_extra["capacidad"] = int(input("Capacidad de personas: "))
                    elif tipo.upper() == "EQUIPO":
                        datos_extra["tipo"] = input("Descripción del equipo: ")
                    elif tipo.upper() == "ASESORIA":
                        datos_extra["especialidad"] = input("Especialidad: ")

                    self.crear_servicio(tipo, nombre, precio, **datos_extra)

                elif opcion == 6:
                    self.mostrar_servicios()

                elif opcion == 7:
                    print("\n--- EDITAR SERVICIO ---")
                    id_servicio = int(input("ID del servicio a editar: "))
                    nombre = input("Nuevo nombre del servicio: ")
                    precio = float(input("Nuevo precio base ($): "))
                    self.editar_servicio(id_servicio, nombre, precio)

                elif opcion == 8:
                    print("\n--- ELIMINAR SERVICIO ---")
                    id_servicio = int(input("ID del servicio a eliminar: "))
                    confirmacion = input(f"¿Confirma eliminar servicio {id_servicio}? (s/n): ")
                    if confirmacion.lower() == "s":
                        self.eliminar_servicio(id_servicio)
                    else:
                        print("❌ Operación cancelada")

                elif opcion == 9:
                    print("\n--- CREAR RESERVA ---")
                    id_cliente = int(input("ID del cliente: "))
                    id_servicio = int(input("ID del servicio: "))
                    duracion = float(input("Duración en horas: "))
                    self.crear_reserva(id_cliente, id_servicio, duracion)

                elif opcion == 10:
                    self.mostrar_reservas()

                elif opcion == 11:
                    print("\n--- EDITAR RESERVA ---")
                    id_reserva = int(input("ID de la reserva a editar: "))
                    duracion = float(input("Nueva duración en horas: "))
                    self.editar_reserva(id_reserva, duracion)

                elif opcion == 12:
                    print("\n--- CONFIRMAR RESERVA ---")
                    id_reserva = int(input("ID de la reserva: "))
                    print("¿Desea aplicar descuento? (0.1 = 10%)")
                    desc = float(input("Descuento (0 si no quiere): "))
                    self.confirmar_reserva(id_reserva, descuento=desc)

                elif opcion == 13:
                    print("\n--- CANCELAR RESERVA ---")
                    id_reserva = int(input("ID de la reserva: "))
                    self.cancelar_reserva(id_reserva)

                elif opcion == 14:
                    print("\n--- ELIMINAR RESERVA ---")
                    id_reserva = int(input("ID de la reserva a eliminar: "))
                    confirmacion = input(f"¿Confirma eliminar reserva {id_reserva}? (s/n): ")
                    if confirmacion.lower() == "s":
                        self.eliminar_reserva(id_reserva)
                    else:
                        print("❌ Operación cancelada")

                elif opcion == 15:
                    print("👋 Saliendo del sistema... Registro guardado en 'registro_logs.txt'")
                    GestorLog.registrar("SALIDA", "Usuario finalizó la ejecución del sistema")
                    break

                else:
                    print("⚠️ Opción inválida. Ingrese un número entre 1 y 15.")

            except ValueError:
                print("❌ Error: Debe ingresar un número válido.")
                GestorLog.registrar("ERROR", "Entrada no numérica en menú principal")
            except Exception as e:
                print(f"⚠️ Error inesperado: {str(e)}")
                GestorLog.registrar("ERROR", f"Error en menú: {str(e)}")
                print("Intente nuevamente...")

    # ---------- PRUEBAS AUTOMÁTICAS (REQUISITO TAREA) ----------
    def ejecutar_pruebas_automaticas(self):
        """Ejecuta mínimo 10 operaciones para demostrar funcionamiento"""
        print("\n🔎 EJECUTANDO PRUEBAS AUTOMÁTICAS DEL SISTEMA...")
        try:
            # 1. Registrar clientes válidos
            self.registrar_cliente("María López", "maria@correo.com", "3101234567")
            self.registrar_cliente("Carlos Ruiz", "carlos@correo.com", "3207654321")
            # 2. Intentar cliente inválido
            self.registrar_cliente("An", "correo-malo", "123")

            # 3. Crear servicios
            self.crear_servicio("SALA", "Sala de Conferencias", 80.0, capacidad=50)
            self.crear_servicio("EQUIPO", "Proyector 4K", 25.0, tipo="Video")
            self.crear_servicio("ASESORIA", "Asesoría Legal", 120.0, especialidad="Derecho")

            # 4. Crear reservas
            self.crear_reserva(1000, 2000, 3)
            self.crear_reserva(1001, 2001, 2)
            # 5. Reserva inválida (duración negativa)
            self.crear_reserva(1000, 2002, -1)

            # 6. Confirmar reserva
            self.confirmar_reserva(3000, descuento=0.1)
            # 7. Cancelar reserva
            self.cancelar_reserva(3001)

            print("✅ Pruebas automáticas finalizadas. Revisa el archivo de log.")
            GestorLog.registrar("PRUEBA", "Ejecución de 10+ operaciones de prueba completada")

        except Exception as e:
            print(f"⚠️ Error en pruebas: {e}")


# ==============================================
# 5. EJECUCIÓN PRINCIPAL DEL PROGRAMA
# ==============================================
if __name__ == "__main__":
    # Inicializar sistema
    sistema = SistemaGestion()
    GestorLog.registrar("INICIO", "Sistema iniciado correctamente")

    # Ejecutar pruebas automáticas 
    sistema.ejecutar_pruebas_automaticas()

    # Iniciar menú interactivo
    sistema.menu_principal()
