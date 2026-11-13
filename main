"""ReservaGol | Gestión de alquiler de canchas (Tkinter + servidor PHP/MySQL).

El programa no guarda datos en tu computadora: todo se consulta y se guarda en el
servidor (carpeta `servidor`). Antes de abrirlo, instalá el servidor y revisá API_URL.
"""

import json
import os
import urllib.error
import urllib.request
from dataclasses import dataclass
from datetime import date, datetime, timedelta
from pathlib import Path
from typing import Any
import tkinter as tk
from tkinter import messagebox, ttk

# ───────────────────────── Configuración ─────────────────────────

APP_TITLE = "ReservaGol | Gestión de canchas"
# Dirección del servidor. Se puede cambiar acá o con la variable de entorno RESERVAGOL_API.
API_URL = os.environ.get("RESERVAGOL_API", "http://localhost/reservagol/api.php")
BASE_DIR = Path(__file__).resolve().parent  # los logos viven junto a este archivo

FONT = "Segoe UI"
BG = "#eef2f7"          # fondo general
FORM_BG = "#f8fafc"     # fondo de formularios
GREEN_DARK = "#123b32"  # barra superior y títulos
GREEN = "#087f5b"       # acento principal
WHITE = "white"
TEXT = "#344054"
MUTED = "#667085"
BORDER = "#d0d5dd"

DIAS = ["Lunes", "Martes", "Miércoles", "Jueves", "Viernes", "Sábado", "Domingo"]
METODOS_PAGO = ["Efectivo", "Transferencia", "Tarjeta", "Mercado Pago"]
ESTADOS_CANCHA = ["Disponible", "Mantenimiento"]
CONCEPTOS_PAGO = ["Seña", "Pago"]
SI_NO = [("1", "Sí"), ("0", "No")]
ROLES = {"admin": "Administrador", "empleado": "Empleado", "cliente": "Cliente"}

# Tarjetas de la pantalla de inicio: (dato del servidor, título, color)
STATS = [
    ("canchas", "Canchas registradas", GREEN),
    ("reservas", "Reservas confirmadas", "#175cd3"),
    ("clientes", "Clientes registrados", "#7f56d9"),
]


# ───────────────────────── Funciones auxiliares ─────────────────────────

def money(value):
    """18000 -> '$ 18.000'"""
    return f"$ {float(value or 0):,.0f}".replace(",", ".")


def hhmm(time_text):
    """'17:00:00' -> '17:00'"""
    return str(time_text)[:5]


def label(parent, text, size=10, bold=False, color=TEXT, bg=BG):
    return tk.Label(parent, text=text, bg=bg, fg=color,
                    font=(FONT, size, "bold" if bold else "normal"))


def add_field(parent, text, widget, bg=BG):
    """Etiqueta + campo apilados verticalmente."""
    label(parent, text, bg=bg).pack(anchor="w")
    widget.pack(fill="x", pady=(4, 12), ipady=4)
    return widget


def load_logo(root, filename):
    """Carga un logo PNG. Si el archivo no está, devuelve None y se usa texto."""
    try:
        return tk.PhotoImage(master=root, file=str(BASE_DIR / filename))
    except tk.TclError:
        return None


def make_tree(parent, columns, **options):
    """Crea una tabla. columns = [(clave, título, ancho), ...]"""
    tree = ttk.Treeview(parent, columns=[c[0] for c in columns], show="headings", **options)
    for key, title, width in columns:
        tree.heading(key, text=title)
        tree.column(key, width=width, anchor="w")
    return tree


def fill_tree(tree, rows):
    """Reemplaza el contenido de una tabla. rows = [(id, valores), ...]"""
    tree.delete(*tree.get_children())
    for iid, values in rows:
        tree.insert("", "end", iid=str(iid), values=values)


def selected_id(tree, message):
    """Devuelve el id seleccionado en la tabla, o avisa si no hay ninguno."""
    selection = tree.selection()
    if not selection:
        messagebox.showinfo("Seleccioná un elemento", message)
        return None
    return int(selection[0])


def cell_value(row, column):
    """Texto de una celda. column = (clave, título, ancho) o (clave, título, ancho, formato)."""
    key, _title, _width, *fmt = column
    value = row.get(key)
    if value is None:
        return ""
    return fmt[0](value) if fmt else value


def payment_status(row):
    """Estado de pago de una reserva, según lo cobrado, la seña pedida y el total."""
    if row["estado"] != "Confirmada":
        return "—"
    paid, total, deposit = float(row["pagado"]), float(row["monto_total"]), float(row["sena_requerida"])
    if paid + 0.005 >= total:
        return "Pagada"
    if deposit > 0 and paid + 0.005 >= deposit:
        return "Seña paga"
    return "Pago parcial" if paid > 0 else "Sin pagar"


# ───────────────────────── Conexión con el servidor ─────────────────────────

class ApiError(Exception):
    """Error devuelto por el servidor, o problema para conectarse con él."""

    def __init__(self, message, status=0):
        super().__init__(message)
        self.status = status


class Api:
    """Envía pedidos al servidor PHP y devuelve sus respuestas."""

    def __init__(self, url):
        self.url = url
        self.token = None  # se completa al iniciar sesión

    def call(self, action, data=None, timeout=10):
        headers = {"Content-Type": "application/json"}
        if self.token:
            headers["X-Token"] = self.token
        request = urllib.request.Request(
            f"{self.url}?action={action}",
            data=json.dumps(data or {}).encode("utf-8"),
            headers=headers, method="POST")

        status = 200
        try:
            with urllib.request.urlopen(request, timeout=timeout) as response:
                body = response.read()
        except urllib.error.HTTPError as error:  # el servidor respondió con un error
            status, body = error.code, error.read()
        except (urllib.error.URLError, TimeoutError, OSError):
            raise ApiError("No se pudo conectar con el servidor. Revisá que esté encendido "
                           f"y que la dirección sea correcta:\n{self.url}") from None

        try:
            answer = json.loads(body.decode("utf-8"))
        except ValueError:
            raise ApiError("El servidor respondió algo inesperado. "
                           "Revisá que API_URL apunte al archivo api.php.", status) from None
        if not answer.get("ok"):
            raise ApiError(answer.get("error", "Error desconocido del servidor."), status)
        return answer["data"]


# ───────────────────────── Formularios genéricos ─────────────────────────

@dataclass
class Field:
    key: str                # nombre del dato en el servidor
    label: str              # etiqueta que ve la persona
    default: str = ""
    choices: Any = None     # None = texto libre | lista de textos | función que devuelve [(valor, texto), ...]
    hidden: bool = False    # oculta lo escrito (contraseñas)


class CrudPanel:
    """Formulario + tabla para crear, editar y eliminar un tipo de registro del servidor."""

    def __init__(self, parent, app, entity, id_key, fields, columns, noun,
                 name_of=lambda row: row["nombre"], warn_of=lambda row: "", per_row=4):
        self.app, self.entity, self.id_key, self.noun = app, entity, id_key, noun
        self.fields, self.columns = fields, columns
        self.name_of, self.warn_of, self.per_row = name_of, warn_of, per_row
        self.rows = {}            # id -> fila, tal como la devolvió el servidor
        self.editing_id = None    # id del registro que se está editando (None = uno nuevo)
        self.widgets = {}         # clave -> campo del formulario
        self.choice_values = {}   # clave -> {texto que se ve: valor que se envía}
        self.loaded = False
        self.build(parent)

    def build(self, parent):
        form = tk.Frame(parent, bg=FORM_BG, padx=10, pady=6)
        form.pack(fill="x", padx=10, pady=(10, 4))
        for i, field in enumerate(self.fields):
            row, col = divmod(i, self.per_row)
            cell = tk.Frame(form, bg=FORM_BG)
            cell.grid(row=row, column=col, sticky="ew", padx=6, pady=3)
            form.grid_columnconfigure(col, weight=1)
            label(cell, field.label, 9, True, bg=FORM_BG).pack(anchor="w")
            if field.choices is None:
                widget = ttk.Entry(cell, show="•" if field.hidden else "")
            else:
                widget = ttk.Combobox(cell, state="readonly")
            widget.pack(fill="x", pady=(3, 0))
            self.widgets[field.key] = widget

        buttons = tk.Frame(parent, padx=10)
        buttons.pack(fill="x", pady=(0, 2))
        ttk.Button(buttons, text="Guardar", style="Primary.TButton",
                   command=self.save).pack(side="left", padx=(0, 8))
        ttk.Button(buttons, text="Limpiar formulario", command=self.clear_form).pack(side="left")
        ttk.Button(buttons, text="Eliminar seleccionado", command=self.delete).pack(side="right")

        self.tree = make_tree(parent, [c[:3] for c in self.columns], height=4)
        self.tree.pack(fill="both", expand=True, padx=10, pady=(6, 8))
        self.tree.bind("<<TreeviewSelect>>", self.load_selected)
        self.tree.bind("<Delete>", lambda _e: self.delete())

    # --- datos ---

    def load_choices(self):
        """Actualiza las opciones de los campos desplegables."""
        for field in self.fields:
            if field.choices is None:
                continue
            if callable(field.choices):
                options = field.choices()
            else:  # lista de textos, o de pares (valor, texto)
                options = [(o, o) if isinstance(o, str) else o for o in field.choices]
            self.choice_values[field.key] = {text: value for value, text in options}
            self.widgets[field.key]["values"] = list(self.choice_values[field.key])

    def refresh(self):
        self.load_choices()
        rows = self.app.request(f"{self.entity}.list")
        if rows is None:
            return
        self.rows = {int(row[self.id_key]): row for row in rows}
        fill_tree(self.tree, [(row_id, tuple(cell_value(row, c) for c in self.columns))
                              for row_id, row in self.rows.items()])
        if not self.loaded:  # la primera vez deja el formulario con sus valores iniciales
            self.clear_form()
            self.loaded = True

    # --- formulario ---

    def set_widget(self, field, value):
        widget = self.widgets[field.key]
        if field.choices is None:
            widget.delete(0, "end")
            widget.insert(0, "" if value is None else str(value))
        else:
            shown = [text for text, v in self.choice_values.get(field.key, {}).items()
                     if str(v) == str(value)]
            widget.set(shown[0] if shown else "")

    def clear_form(self):
        self.editing_id = None
        for field in self.fields:
            self.set_widget(field, field.default)

    def load_selected(self, _event=None):
        selection = self.tree.selection()
        if not selection:
            return
        row = self.rows.get(int(selection[0]))
        if row is None:
            return
        self.editing_id = int(selection[0])
        for field in self.fields:
            self.set_widget(field, row.get(field.key))

    def read_form(self):
        data = {}
        for field in self.fields:
            widget = self.widgets[field.key]
            text = widget.get() if field.hidden else widget.get().strip()
            if field.choices is not None:
                text = self.choice_values.get(field.key, {}).get(text, "")
            data[field.key] = text
        return data

    # --- acciones ---

    def save(self):
        data = self.read_form()
        if self.editing_id:
            data["id"] = self.editing_id
        if self.app.request(f"{self.entity}.save", data) is None:
            return
        self.clear_form()
        self.refresh()
        messagebox.showinfo("Guardado", "Los datos se guardaron correctamente.")

    def delete(self):
        row_id = selected_id(self.tree, f"Elegí qué {self.noun} querés eliminar.")
        if row_id is None or row_id not in self.rows:
            return
        if not messagebox.askyesno(f"Eliminar {self.noun}",
                                   f'¿Seguro que querés eliminar "{self.name_of(self.rows[row_id])}"?'
                                   + self.warn_of(self.rows[row_id])):
            return
        if self.app.request(f"{self.entity}.delete", {"id": row_id}) is None:
            return
        self.clear_form()
        self.refresh()


class ListPanel:
    """Tabla de solo lectura con un total al pie."""

    def __init__(self, parent, app, action, id_key, columns, total_key, total_text):
        self.app, self.action, self.id_key = app, action, id_key
        self.columns, self.total_key, self.total_text = columns, total_key, total_text
        self.total = label(parent, "", 11, True, GREEN_DARK)
        self.total.pack(side="bottom", anchor="e", padx=12, pady=(0, 8))  # primero, para que la tabla no lo tape
        self.tree = make_tree(parent, [c[:3] for c in columns], height=4)
        self.tree.pack(fill="both", expand=True, padx=10, pady=(10, 6))

    def refresh(self):
        rows = self.app.request(self.action)
        if rows is None:
            return
        fill_tree(self.tree, [(row[self.id_key], tuple(cell_value(row, c) for c in self.columns)) for row in rows])
        total = sum(float(row[self.total_key]) for row in rows)
        self.total.config(text=f"{self.total_text}{money(total)}")


class SchedulePanel:
    """Horarios de cada tipo de cancha: se generan por día y se pueden habilitar o quitar."""

    def __init__(self, parent, app):
        self.app = app
        self.types = {}  # texto que se ve -> id del tipo de cancha
        self.rows = {}

        top = tk.Frame(parent, bg=FORM_BG, padx=10, pady=6)
        top.pack(fill="x", padx=10, pady=(10, 4))
        self.type_box = ttk.Combobox(top, state="readonly", width=30)
        self.day_box = ttk.Combobox(top, state="readonly", width=14, values=["Todos los días"] + DIAS)
        self.day_box.set("Todos los días")
        self.start = ttk.Entry(top, width=8)
        self.start.insert(0, "17:00")
        self.end = ttk.Entry(top, width=8)
        self.end.insert(0, "23:00")
        self.minutes = ttk.Combobox(top, state="readonly", width=8, values=["60", "90", "120"])
        self.minutes.set("60")
        fields = [("Tipo de cancha", self.type_box), ("Día", self.day_box), ("Desde (HH:MM)", self.start),
                  ("Hasta (HH:MM)", self.end), ("Duración (min)", self.minutes)]
        for col, (text, widget) in enumerate(fields):
            label(top, text, 9, True, bg=FORM_BG).grid(row=0, column=col, sticky="w", padx=6)
            widget.grid(row=1, column=col, sticky="ew", padx=6)
        ttk.Button(top, text="Generar turnos", style="Primary.TButton",
                   command=self.generate).grid(row=1, column=len(fields), padx=6)
        self.type_box.bind("<<ComboboxSelected>>", lambda _e: self.load_slots())

        buttons = tk.Frame(parent, padx=10)
        buttons.pack(fill="x", pady=(0, 2))
        ttk.Button(buttons, text="Habilitar / deshabilitar horario", command=self.toggle).pack(side="left")
        ttk.Button(buttons, text="Eliminar horario", command=self.delete).pack(side="right")

        self.tree = make_tree(parent, [("dia", "Día", 130), ("desde", "Desde", 100),
                                       ("hasta", "Hasta", 100), ("estado", "Estado", 120)], height=4)
        self.tree.pack(fill="both", expand=True, padx=10, pady=(6, 8))
        self.tree.bind("<Delete>", lambda _e: self.delete())

    def refresh(self):
        types = self.app.request("tipos.list")
        if types is None:
            return
        self.types = {t["nombre"]: int(t["id_tipo"]) for t in types}
        self.type_box["values"] = list(self.types)
        if self.type_box.get() not in self.types:
            self.type_box.set(next(iter(self.types), ""))
        self.load_slots()

    def selected_type(self):
        type_id = self.types.get(self.type_box.get())
        if type_id is None:
            messagebox.showinfo("Elegí un tipo de cancha", "Primero elegí el tipo de cancha en el desplegable.")
        return type_id

    def load_slots(self):
        type_id = self.types.get(self.type_box.get())
        if type_id is None:
            fill_tree(self.tree, [])
            return
        rows = self.app.request("horarios.list", {"id_tipo": type_id})
        if rows is None:
            return
        self.rows = {int(r["id_horario"]): r for r in rows}
        fill_tree(self.tree, [
            (rid, (r["dia"], hhmm(r["hora_inicio"]), hhmm(r["hora_fin"]),
                   "Habilitado" if int(r["disponible"]) else "Deshabilitado"))
            for rid, r in self.rows.items()])

    def generate(self):
        type_id = self.selected_type()
        if type_id is None:
            return
        day = self.day_box.get()
        result = self.app.request("horarios.generate", {
            "id_tipo": type_id,
            "dia": "Todos" if day == "Todos los días" else day,
            "hora_inicio": self.start.get().strip(),
            "hora_fin": self.end.get().strip(),
            "minutos": int(self.minutes.get() or 60),
        })
        if result is None:
            return
        self.load_slots()
        messagebox.showinfo("Horarios generados", f'Se crearon {result["creados"]} horarios nuevos.')

    def toggle(self):
        slot_id = selected_id(self.tree, "Elegí un horario de la lista.")
        if slot_id is not None and self.app.request("horarios.toggle", {"id": slot_id}) is not None:
            self.load_slots()

    def delete(self):
        slot_id = selected_id(self.tree, "Elegí un horario de la lista.")
        if slot_id is None or not messagebox.askyesno("Eliminar horario", "¿Seguro que querés eliminar este horario?"):
            return
        if self.app.request("horarios.delete", {"id": slot_id}) is not None:
            self.load_slots()


# ───────────────────────── Aplicación ─────────────────────────

class ReservaGolApp:
    def __init__(self, root):
        self.root = root
        self.root.title(APP_TITLE)
        self.root.geometry("1080x700")
        self.root.minsize(900, 600)
        self.api = Api(API_URL)
        self.current_user = None
        self.refreshers = {}  # pestaña -> función que vuelve a cargar sus datos
        self.logo_login = load_logo(root, "logo_login.png")    # logo completo
        self.logo_header = load_logo(root, "logo_header.png")  # solo el emblema
        logo_icon = load_logo(root, "logo_icon.png")
        if logo_icon:
            root.iconphoto(True, logo_icon)
        self.setup_styles()
        self.show_login()

    @property
    def is_admin(self):
        return self.current_user["rol"] == "admin"

    @property
    def is_staff(self):
        return self.current_user["rol"] in ("admin", "empleado")

    def setup_styles(self):
        style = ttk.Style()
        try:
            style.theme_use("clam")
        except tk.TclError:
            pass
        style.configure("TNotebook", background=BG, borderwidth=0)
        style.configure("TNotebook.Tab", padding=(18, 10), font=(FONT, 10, "bold"))
        style.configure("Treeview", rowheight=30, font=(FONT, 10))
        style.configure("Treeview.Heading", font=(FONT, 10, "bold"))
        style.configure("TButton", padding=(10, 7), font=(FONT, 10))
        style.configure("Primary.TButton", background=GREEN, foreground=WHITE)
        style.map("Primary.TButton", background=[("active", "#066649")])

    def clear(self):
        for widget in self.root.winfo_children():
            widget.destroy()

    def request(self, action, data=None, title="No se pudo completar la acción", parent=None):
        """Llama al servidor. Si algo falla muestra el error y devuelve None."""
        try:
            return self.api.call(action, data)
        except ApiError as error:
            if error.status == 401 and self.current_user:  # la sesión venció
                self.api.token = self.current_user = None
                messagebox.showwarning("Sesión vencida", str(error))
                self.show_login()
            else:
                messagebox.showerror(title, str(error), parent=parent)
            return None

    # ───────────── Acceso: inicio de sesión y registro ─────────────

    def auth_card(self, subtitle, width=344):
        """Logo sobre el fondo verde y tarjeta blanca debajo, común a login y registro."""
        self.clear()
        self.root.configure(bg=GREEN_DARK)
        box = tk.Frame(self.root, bg=GREEN_DARK)
        box.place(relx=0.5, rely=0.5, anchor="center")
        if self.logo_login:
            tk.Label(box, image=self.logo_login, bg=GREEN_DARK).pack(pady=(0, 14))
        else:
            label(box, "⚽  ReservaGol", 25, True, "white", GREEN_DARK).pack(pady=(0, 14))
        card = tk.Frame(box, bg=WHITE, padx=38, pady=28)
        card.pack()
        tk.Frame(card, bg=WHITE, width=width, height=1).pack()  # fija el ancho de la tarjeta
        label(card, subtitle, color=MUTED, bg=WHITE).pack(pady=(0, 18))
        return card

    @staticmethod
    def card_field(card, text, hidden=False):
        label(card, text, bold=True, bg=WHITE).pack(anchor="w")
        entry = ttk.Entry(card, font=(FONT, 11), show="•" if hidden else "")
        entry.pack(fill="x", ipady=6, pady=(5, 14))
        return entry

    def show_login(self, username=""):
        card = self.auth_card("Gestión de alquileres de canchas")
        self.login_user = self.card_field(card, "Usuario")
        self.login_password = self.card_field(card, "Contraseña", hidden=True)
        self.login_password.bind("<Return>", lambda _e: self.login())
        ttk.Button(card, text="Iniciar sesión", style="Primary.TButton",
                   command=self.login).pack(fill="x", ipady=3)

        # Registro de cuenta, debajo de los datos de acceso
        ttk.Separator(card).pack(fill="x", pady=16)
        label(card, "¿Todavía no tenés cuenta?", color=MUTED, bg=WHITE).pack(pady=(0, 6))
        ttk.Button(card, text="Registrar cuenta", command=self.show_register).pack(fill="x", ipady=3)

        status = label(card, "Conectando con el servidor…", 9, color=MUTED, bg=WHITE)
        status.pack(pady=(14, 0))
        self.root.after(100, lambda: self.check_server(status))

        if username:  # viene de registrarse: dejamos el usuario cargado
            self.login_user.insert(0, username)
            self.login_password.focus_set()
        else:
            self.login_user.focus_set()

    def check_server(self, status):
        """Muestra si el servidor responde, para detectar problemas de conexión desde el inicio."""
        try:
            self.api.call("ping", timeout=4)
            text, color = "● Servidor conectado", GREEN
        except ApiError as error:
            text, color = f"● Sin conexión con el servidor\n{str(error).splitlines()[0]}", "#b42318"
        try:
            status.config(text=text, fg=color)
        except tk.TclError:  # la pantalla ya cambió mientras se esperaba la respuesta
            pass

    def show_register(self):
        card = self.auth_card("Creá tu cuenta de cliente", width=484)
        form = tk.Frame(card, bg=WHITE)
        form.pack(fill="x")
        for col in range(6):
            form.grid_columnconfigure(col, weight=1, uniform="registro")

        # (dato, etiqueta, fila, columna, ancho, oculto)
        layout = [("nombre", "Nombre", 0, 0, 3, False), ("apellido", "Apellido", 0, 3, 3, False),
                  ("telefono", "Teléfono", 1, 0, 3, False), ("email", "Email (opcional)", 1, 3, 3, False),
                  ("username", "Usuario", 2, 0, 2, False), ("password", "Contraseña", 2, 2, 2, True),
                  ("repeat", "Repetir contraseña", 2, 4, 2, True)]
        entries = {}
        for key, text, row, col, span, hidden in layout:
            cell = tk.Frame(form, bg=WHITE)
            cell.grid(row=row, column=col, columnspan=span, sticky="ew", padx=4)
            label(cell, text, 9, True, bg=WHITE).pack(anchor="w")
            entry = ttk.Entry(cell, font=(FONT, 10), show="•" if hidden else "")
            entry.pack(fill="x", ipady=4, pady=(4, 12))
            entries[key] = entry

        def submit():
            data = {key: entry.get() if key in ("password", "repeat") else entry.get().strip()
                    for key, entry in entries.items()}
            if data.pop("repeat") != data["password"]:
                messagebox.showerror("Contraseñas distintas", "Las contraseñas no coinciden.")
                return
            if self.request("register", data, title="No se pudo registrar") is None:
                return
            messagebox.showinfo("Cuenta creada",
                                f"Tu cuenta '{data['username']}' fue creada. Ya podés iniciar sesión.")
            self.show_login(data["username"])

        entries["repeat"].bind("<Return>", lambda _e: submit())
        ttk.Button(card, text="Crear cuenta", style="Primary.TButton",
                   command=submit).pack(fill="x", ipady=3)
        ttk.Button(card, text="Volver a iniciar sesión",
                   command=self.show_login).pack(fill="x", ipady=3, pady=(10, 0))
        entries["nombre"].focus_set()

    def login(self):
        data = self.request("login", {"username": self.login_user.get().strip(),
                                      "password": self.login_password.get()},
                            title="No se pudo iniciar sesión")
        if data is None:
            return
        self.api.token = data["token"]
        self.current_user = data["usuario"]
        self.show_dashboard()

    def logout(self):
        try:
            self.api.call("logout")
        except ApiError:
            pass  # si no hay conexión, igual se cierra la sesión en este equipo
        self.api.token = self.current_user = None
        self.show_login()

    # ───────────── Panel principal ─────────────

    def show_dashboard(self):
        self.clear()
        self.refreshers = {}
        self.root.configure(bg=BG)

        top = tk.Frame(self.root, bg=GREEN_DARK, height=76)
        top.pack(fill="x")
        top.pack_propagate(False)
        brand = tk.Frame(top, bg=GREEN_DARK)
        brand.pack(side="left", padx=24)
        if self.logo_header:
            tk.Label(brand, image=self.logo_header, bg=GREEN_DARK).pack(side="left", padx=(0, 12))
        label(brand, "ReservaGol" if self.logo_header else "⚽  ReservaGol",
              20, True, "white", GREEN_DARK).pack(side="left")
        label(top, f'{self.current_user["username"]}  ·  {ROLES[self.current_user["rol"]]}',
              color="#d0e7df", bg=GREEN_DARK).pack(side="right", padx=22)
        ttk.Button(top, text="Cerrar sesión", command=self.logout).pack(side="right", padx=8, pady=18)

        self.notebook = ttk.Notebook(self.root)
        self.notebook.pack(fill="both", expand=True, padx=18, pady=18)
        self.notebook.bind("<<NotebookTabChanged>>", self.on_tab_change)
        self.build_home_tab()
        self.build_courts_tab()
        self.build_reservations_tab()
        if self.is_staff:
            self.build_admin_tab()
        self.root.after(50, lambda: self.refresh_selected(self.notebook))

    def new_tab(self, tab_name, title, subtitle, refresh):
        """Agrega una pestaña con título y subtítulo, y devuelve su contenedor."""
        tab = ttk.Frame(self.notebook)
        self.notebook.add(tab, text=tab_name)
        self.refreshers[str(tab)] = refresh
        frame = tk.Frame(tab, bg=BG, padx=18, pady=18)
        frame.pack(fill="both", expand=True)
        label(frame, title, 19, True, GREEN_DARK).pack(anchor="w")
        label(frame, subtitle, color=MUTED).pack(anchor="w", pady=(4, 14))
        return frame

    def on_tab_change(self, event):
        self.refresh_selected(event.widget)

    def refresh_selected(self, notebook):
        """Vuelve a cargar los datos de la pestaña que se está mirando."""
        refresh = self.refreshers.get(str(notebook.select()))
        if refresh:
            refresh()

    # ───────────── Pestaña: Inicio ─────────────

    def build_home_tab(self):
        frame = self.new_tab("Inicio", "Panel principal",
                             "Resumen general del sistema de alquiler de canchas.",
                             refresh=self.refresh_stats)
        stats = tk.Frame(frame, bg=BG)
        stats.pack(fill="x", pady=8)
        self.stat_values = {}
        for key, title, color in STATS:
            card = tk.Frame(stats, bg=WHITE, padx=18, pady=17,
                            highlightbackground=BORDER, highlightthickness=1)
            card.pack(side="left", fill="x", expand=True, padx=6)
            label(card, title, color=MUTED, bg=WHITE).pack(anchor="w")
            value = label(card, "0", 25, True, color, WHITE)
            value.pack(anchor="w", pady=(6, 0))
            self.stat_values[key] = value

        info = tk.Frame(frame, bg=WHITE, padx=20, pady=18,
                        highlightbackground=BORDER, highlightthickness=1)
        info.pack(fill="x", pady=18)
        label(info, "¿Qué podés hacer en ReservaGol?", 13, True, GREEN_DARK, WHITE).pack(anchor="w", pady=(0, 10))
        lines = ["• Canchas: consultar cada cancha, su tipo, superficie, capacidad y precio.",
                 "• Reservas: elegir cancha, fecha y horario; pagar la seña, registrar pagos y cancelar."]
        if self.is_staff:
            lines.append("• Administración: tipos de cancha, canchas, horarios, usuarios y facturación.")
        for line in lines:
            label(info, line, color="#475467", bg=WHITE).pack(anchor="w", pady=3)

        if self.is_staff:
            role = "administrador" if self.is_admin else "empleado"
            tk.Label(frame, text=f"Tenés permisos de {role}: podés gestionar canchas, horarios, usuarios y todas las reservas.",
                     bg="#e0f2e9", fg="#05603a", padx=12, pady=10,
                     font=(FONT, 10, "bold")).pack(anchor="w", fill="x")

    def refresh_stats(self):
        stats = self.request("stats")
        if stats is None:
            return
        for key, value in self.stat_values.items():
            value.config(text=str(stats.get(key, 0)))

    # ───────────── Pestaña: Canchas ─────────────

    def build_courts_tab(self):
        frame = self.new_tab("Canchas", "Canchas disponibles",
                             "Elegí una cancha y consultá su tipo, superficie, capacidad y precio por hora.",
                             refresh=self.refresh_courts)
        actions = tk.Frame(frame, bg=BG)
        actions.pack(side="bottom", fill="x")  # primero, para que la tabla nunca lo tape
        ttk.Button(actions, text="Actualizar", command=self.refresh_courts).pack(side="left")
        ttk.Button(actions, text="Reservar cancha seleccionada", style="Primary.TButton",
                   command=self.open_booking).pack(side="right")

        self.courts = {}
        self.courts_tree = make_tree(frame, [
            ("nombre", "Cancha", 110), ("tipo", "Tipo", 90), ("superficie", "Superficie", 130),
            ("capacidad", "Jugadores", 75), ("precio_hora", "Precio por hora", 115), ("sena", "Seña", 60),
            ("extras", "Características", 150), ("ubicacion", "Ubicación", 110), ("estado", "Estado", 100),
        ], selectmode="browse")
        self.courts_tree.pack(fill="both", expand=True, pady=(0, 12))
        self.courts_tree.bind("<Double-1>", lambda _e: self.open_booking())

    def refresh_courts(self):
        rows = self.request("canchas.list")
        if rows is None:
            return
        self.courts = {int(r["id_cancha"]): r for r in rows}
        fill_tree(self.courts_tree, [
            (cid, (c["nombre"], c["tipo"], c["superficie"], c["capacidad"], money(c["precio_hora"]),
                   f'{float(c["porcentaje_sena"]):g} %', self.features(c), c["ubicacion"] or "",
                   c["estado"] if c["estado_tipo"] == "Disponible" else "No disponible"))
            for cid, c in self.courts.items()])

    @staticmethod
    def features(court):
        """Texto corto con las características del tipo de cancha."""
        parts = ["Techada" if int(court["techada"]) else "Descubierta"]
        parts.append("Con luz" if int(court["iluminacion"]) else "Sin luz")
        return ", ".join(parts)

    # ───────────── Nueva reserva ─────────────

    def open_booking(self):
        court_id = selected_id(self.courts_tree, "Primero seleccioná una cancha de la lista.")
        if court_id is None:
            return
        court = self.courts[court_id]

        clients = {}
        if self.is_staff:  # el personal reserva en nombre de un cliente
            rows = self.request("usuarios.list")
            if rows is None:
                return
            clients = {f'{u["apellido"]}, {u["nombre"]}': int(u["id_usuario"])
                       for u in rows if u["rol"] == "cliente" and int(u["activo"])}
            if not clients:
                messagebox.showinfo("Faltan clientes",
                                    "Para reservar en nombre de un cliente, primero tiene que existir uno.\n\n"
                                    "Creá la cuenta en Administración → Usuarios (rol Cliente), o pedile que se "
                                    "registre desde la pantalla de inicio de sesión.")
                return

        win = tk.Toplevel(self.root)
        win.title("Nueva reserva")
        win.geometry("440x570" if self.is_staff else "440x500")
        win.resizable(False, False)
        win.transient(self.root)
        win.grab_set()
        box = tk.Frame(win, padx=24, pady=20, bg=BG)
        box.pack(fill="both", expand=True)

        label(box, "Nueva reserva", 18, True, GREEN_DARK).pack(anchor="w")
        label(box, f'{court["nombre"]} ({court["tipo"]}) · {money(court["precio_hora"])}/hora · seña {float(court["porcentaje_sena"]):g} %',
              color="#475467").pack(anchor="w", pady=(5, 16))

        client_box = None
        if self.is_staff:
            client_box = add_field(box, "Cliente", ttk.Combobox(box, state="readonly", values=list(clients)))
        date_entry = add_field(box, "Fecha (AAAA-MM-DD)", ttk.Entry(box))
        date_entry.insert(0, date.today().isoformat())
        slot_box = add_field(box, "Horario libre", ttk.Combobox(box, state="readonly"))
        slots = {}
        hint = label(box, "", color=MUTED)
        hint.config(wraplength=380, justify="left")

        def load_slots(search_ahead=False):
            """Muestra los horarios libres de la fecha. Con search_ahead, salta al próximo día con lugar."""
            slots.clear()
            try:
                day = datetime.strptime(date_entry.get().strip(), "%Y-%m-%d").date()
            except ValueError:
                messagebox.showerror("Fecha inválida", "Escribí la fecha como AAAA-MM-DD.", parent=win)
                return
            moved = False
            for _ in range(15 if search_ahead else 1):
                rows = self.request("horarios.list", {"id_cancha": court_id, "fecha": day.isoformat()}, parent=win)
                if rows is None:
                    return
                if rows or not search_ahead:
                    break
                day += timedelta(days=1)
                moved = True
            if moved:
                date_entry.delete(0, "end")
                date_entry.insert(0, day.isoformat())
            for slot in rows:
                slots[f'{hhmm(slot["hora_inicio"])} - {hhmm(slot["hora_fin"])}'] = int(slot["id_horario"])
            slot_box["values"] = list(slots)
            slot_box.set(next(iter(slots), ""))
            if slots:
                hint.config(text="Hoy ya no quedan horarios: se muestra el próximo día con lugar." if moved else "")
            else:
                hint.config(text="No hay horarios libres para esa fecha. Los horarios de hoy que ya pasaron no se "
                                 "muestran; revisá que el tipo tenga horarios cargados (Administración → Horarios).")

        ttk.Button(box, text="Ver horarios libres", command=load_slots).pack(fill="x")
        hint.pack(anchor="w", pady=(8, 8))

        def confirm():
            if not slot_box.get():
                messagebox.showerror("Falta el horario", "Elegí un horario libre.", parent=win)
                return
            data = {"id_cancha": court_id, "id_horario": slots[slot_box.get()],
                    "fecha": date_entry.get().strip()}
            if client_box is not None:
                if not client_box.get():
                    messagebox.showerror("Falta el cliente", "Elegí el cliente de la reserva.", parent=win)
                    return
                data["id_usuario"] = clients[client_box.get()]
            result = self.request("reservas.create", data, title="No se pudo reservar", parent=win)
            if result is None:
                return
            win.destroy()
            messagebox.showinfo("Reserva confirmada",
                                f'La reserva se guardó. Total: {money(result["monto_total"])}.\n'
                                f'Seña requerida: {money(result["sena_requerida"])}.\n'
                                "Podés registrar el pago desde la pestaña Reservas.")

        ttk.Button(box, text="Confirmar reserva", style="Primary.TButton",
                   command=confirm).pack(fill="x", ipady=3)
        load_slots(search_ahead=True)

    # ───────────── Pestaña: Reservas ─────────────

    def build_reservations_tab(self):
        frame = self.new_tab("Reservas", "Gestión de reservas",
                             "Los clientes ven sus reservas. El personal puede consultar y gestionar todas.",
                             refresh=self.refresh_reservations)
        actions = tk.Frame(frame, bg=BG)
        actions.pack(side="bottom", fill="x")
        ttk.Button(actions, text="Actualizar", command=self.refresh_reservations).pack(side="left")
        ttk.Button(actions, text="Cancelar reserva", command=self.cancel_reservation).pack(side="right")
        ttk.Button(actions, text="Registrar pago o seña", style="Primary.TButton",
                   command=self.open_payment).pack(side="right", padx=8)
        ttk.Button(actions, text="Ver pagos", command=self.show_payments).pack(side="right")

        self.reservations = {}
        self.res_tree = make_tree(frame, [
            ("id", "N.º", 40), ("cliente", "Cliente", 125), ("cancha", "Cancha", 80), ("tipo", "Tipo", 75),
            ("fecha", "Fecha", 85), ("horario", "Horario", 95), ("total", "Total", 80),
            ("pagado", "Pagado", 80), ("pago", "Pago", 85), ("estado", "Estado", 85),
        ], selectmode="browse")
        self.res_tree.pack(fill="both", expand=True, pady=(0, 12))

    def refresh_reservations(self):
        rows = self.request("reservas.list")
        if rows is None:
            return
        self.reservations = {int(r["id_reserva"]): r for r in rows}
        fill_tree(self.res_tree, [
            (rid, (rid, f'{r["cliente_nombre"]} {r["cliente_apellido"]}', r["cancha"], r["tipo"], r["fecha_reserva"],
                   f'{hhmm(r["hora_inicio"])} - {hhmm(r["hora_fin"])}', money(r["monto_total"]),
                   money(r["pagado"]), payment_status(r), r["estado"]))
            for rid, r in self.reservations.items()])

    def selected_reservation(self):
        reservation_id = selected_id(self.res_tree, "Elegí una reserva de la lista.")
        return self.reservations.get(reservation_id) if reservation_id is not None else None

    def cancel_reservation(self):
        row = self.selected_reservation()
        if row is None:
            return
        if row["estado"] != "Confirmada":
            messagebox.showinfo("Reserva", "Esta reserva ya no está confirmada.")
        elif messagebox.askyesno("Confirmar cancelación",
                                 "¿Querés cancelar esta reserva?\n\nSi ya hay pagos, se reintegran solo cuando "
                                 "la cancelación se hace con la anticipación que fija el complejo."):
            result = self.request("reservas.cancel", {"id_reserva": row["id_reserva"]})
            if result is not None:
                messagebox.showinfo("Reserva cancelada", result["mensaje"])
                self.refresh_reservations()

    # ───────────── Facturación: pagos, señas y cancelaciones ─────────────

    def open_payment(self):
        row = self.selected_reservation()
        if row is None:
            return
        total, paid, deposit = float(row["monto_total"]), float(row["pagado"]), float(row["sena_requerida"])
        pending = round(total - paid, 2)
        if row["estado"] != "Confirmada":
            messagebox.showinfo("Pago", "Solo se pueden pagar reservas confirmadas.")
            return
        if pending <= 0:
            messagebox.showinfo("Pago", "Esta reserva ya está paga por completo.")
            return

        win = tk.Toplevel(self.root)
        win.title("Registrar pago o seña")
        win.geometry("420x420")
        win.resizable(False, False)
        win.transient(self.root)
        win.grab_set()
        box = tk.Frame(win, padx=24, pady=20, bg=BG)
        box.pack(fill="both", expand=True)
        label(box, "Registrar pago o seña", 18, True, GREEN_DARK).pack(anchor="w")
        label(box, f'Reserva N.º {row["id_reserva"]} · total {money(total)} · pagado {money(paid)}\n'
                   f'Falta pagar {money(pending)} · seña pedida {money(deposit)}',
              color="#475467", bg=BG).pack(anchor="w", pady=(5, 16))
        concept = add_field(box, "Concepto", ttk.Combobox(box, state="readonly", values=CONCEPTOS_PAGO))
        concept.set("Seña" if paid <= 0 and deposit > 0 else "Pago")
        amount = add_field(box, "Monto ($)", ttk.Entry(box))
        method = add_field(box, "Método de pago", ttk.Combobox(box, state="readonly", values=METODOS_PAGO))
        method.current(0)

        def suggest(_event=None):
            """Propone el monto: la seña pedida, o lo que falta pagar."""
            value = deposit if concept.get() == "Seña" else pending
            amount.delete(0, "end")
            amount.insert(0, f"{min(value, pending):.2f}")

        concept.bind("<<ComboboxSelected>>", suggest)
        suggest()

        def confirm():
            result = self.request("facturacion.create", {
                "id_reserva": row["id_reserva"], "concepto": concept.get(),
                "monto": amount.get().strip(), "metodo_pago": method.get()},
                title="No se pudo registrar el pago", parent=win)
            if result is None:
                return
            win.destroy()
            self.refresh_reservations()
            messagebox.showinfo("Pago registrado", f'Comprobante {result["comprobante"]}. '
                                                   f'Falta pagar {money(result["pendiente"])}.')

        ttk.Button(box, text="Confirmar", style="Primary.TButton", command=confirm).pack(fill="x", ipady=3)

    def show_payments(self):
        row = self.selected_reservation()
        if row is None:
            return
        movements = self.request("facturacion.list", {"id_reserva": row["id_reserva"]})
        if movements is None:
            return
        win = tk.Toplevel(self.root)
        win.title(f'Facturación de la reserva N.º {row["id_reserva"]}')
        win.geometry("760x340")
        win.transient(self.root)
        box = tk.Frame(win, padx=18, pady=16, bg=BG)
        box.pack(fill="both", expand=True)
        label(box, f'Facturación de la reserva N.º {row["id_reserva"]}', 15, True, GREEN_DARK).pack(anchor="w", pady=(0, 10))
        tree = make_tree(box, [("comprobante", "Comprobante", 95), ("fecha", "Fecha", 130), ("concepto", "Concepto", 90),
                               ("monto", "Monto", 90), ("metodo", "Método", 110), ("obs", "Observación", 230)])
        tree.pack(fill="both", expand=True)
        fill_tree(tree, [(m["id_facturacion"], (m["comprobante"] or "", str(m["fecha"])[:16], m["concepto"],
                                                money(m["monto"]), m["metodo_pago"] or "", m["observacion"] or ""))
                         for m in movements])
        if not movements:
            label(box, "Todavía no hay movimientos para esta reserva.", color=MUTED).pack(pady=8)

    # ───────────── Pestaña: Administración ─────────────

    def type_choices(self):
        rows = self.request("tipos.list") or []
        return [(r["id_tipo"], r["nombre"]) for r in rows]

    def build_admin_tab(self):
        frame = self.new_tab("Administración", "Administración del sistema",
                             "Tipos de cancha, canchas, horarios, usuarios y facturación.",
                             refresh=lambda: self.refresh_selected(self.admin_notebook))
        ttk.Button(frame, text="Actualizar", command=lambda: self.refresh_selected(self.admin_notebook)
                   ).pack(side="bottom", anchor="e")
        self.admin_notebook = ttk.Notebook(frame)
        self.admin_notebook.pack(fill="both", expand=True, pady=(0, 12))
        self.admin_notebook.bind("<<NotebookTabChanged>>", self.on_tab_change)

        user_fields = [Field("username", "Usuario"), Field("password", "Contraseña (vacía = sin cambios)", hidden=True)]
        if self.is_admin:  # el empleado solo gestiona clientes, así que no elige rol
            user_fields.append(Field("rol", "Rol", "cliente", [("admin", "Administrador"),
                                                                ("empleado", "Empleado"), ("cliente", "Cliente")]))
        user_fields += [Field("nombre", "Nombre"), Field("apellido", "Apellido"), Field("telefono", "Teléfono"),
                        Field("email", "Email (opcional)"),
                        Field("activo", "Cuenta", "1", [("1", "Activa"), ("0", "Deshabilitada")])]

        def user_warning(row):
            count = int(row.get("reservas") or 0)
            return f"\n\nTambién se eliminarán sus {count} reservas y los pagos de esas reservas." if count else ""

        panels = [
            ("Tipos de cancha", CrudPanel, dict(
                entity="tipos", id_key="id_tipo", noun="tipo de cancha", per_row=5,
                fields=[Field("nombre", "Nombre (Fútbol 5...)"), Field("superficie", "Superficie", "Césped sintético"),
                        Field("medidas", "Medidas", "20 x 40 m"), Field("capacidad", "Jugadores", "10"),
                        Field("techada", "Techada", "0", SI_NO),
                        Field("iluminacion", "Iluminación", "1", SI_NO), Field("precio_hora", "Precio por hora ($)", "18000"),
                        Field("porcentaje_sena", "Seña (%)", "30"),
                        Field("estado", "Estado", "Disponible", ESTADOS_CANCHA), Field("descripcion", "Descripción")],
                columns=[("nombre", "Nombre", 130), ("superficie", "Superficie", 140), ("capacidad", "Jug.", 55),
                         ("precio_hora", "Precio/h", 100, money), ("porcentaje_sena", "Seña %", 70),
                         ("estado", "Estado", 110)])),
            ("Canchas", CrudPanel, dict(
                entity="canchas", id_key="id_cancha", noun="cancha", per_row=5,
                fields=[Field("id_tipo", "Tipo de cancha", choices=self.type_choices),
                        Field("nombre", "Nombre (Cancha 1...)"), Field("ubicacion", "Ubicación", "Planta baja"),
                        Field("estado", "Estado", "Disponible", ESTADOS_CANCHA),
                        Field("observaciones", "Observaciones")],
                columns=[("nombre", "Nombre", 130), ("tipo", "Tipo", 110), ("ubicacion", "Ubicación", 140),
                         ("estado", "Estado", 110), ("observaciones", "Observaciones", 260)])),
            ("Horarios", SchedulePanel, {}),
            ("Usuarios", CrudPanel, dict(
                entity="usuarios", id_key="id_usuario", noun="usuario", fields=user_fields,
                name_of=lambda row: row["username"], warn_of=user_warning,
                columns=[("username", "Usuario", 120), ("rol", "Rol", 100, lambda r: ROLES.get(r, r)),
                         ("nombre", "Nombre", 110), ("apellido", "Apellido", 110), ("telefono", "Teléfono", 100),
                         ("activo", "Cuenta", 100, lambda v: "Activa" if int(v) else "Deshabilitada"),
                         ("reservas", "Reservas", 70)])),
            ("Facturación", ListPanel, dict(
                action="facturacion.list", id_key="id_facturacion",
                total_key="monto", total_text="Total cobrado (neto de reintegros): ",
                columns=[("comprobante", "Comprobante", 95), ("fecha", "Fecha", 130, lambda v: str(v)[:16]),
                         ("id_reserva", "Reserva", 60), ("cliente", "Cliente", 140), ("concepto", "Concepto", 90),
                         ("monto", "Monto", 90, money), ("metodo_pago", "Método", 100), ("observacion", "Observación", 200)])),
        ]
        for name, panel_class, options in panels:
            tab = ttk.Frame(self.admin_notebook)
            self.admin_notebook.add(tab, text=name)
            panel = panel_class(tab, self, **options) if panel_class is not SchedulePanel else panel_class(tab, self)
            self.refreshers[str(tab)] = panel.refresh

    # ───────────── Utilidades ─────────────

    def on_close(self):
        self.root.destroy()


def main():
    root = tk.Tk()
    app = ReservaGolApp(root)
    root.protocol("WM_DELETE_WINDOW", app.on_close)
    root.mainloop()


if __name__ == "__main__":
    main()
