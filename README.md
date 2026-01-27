# AppFilterOfOil
AppFilterOfOil

#CODIGO DE PYEDITER ESE
VOY A INTENTAR CON LAS IMAGENES LLENAR EL INVENTARIO DE TAL MANERA QUE SOLO SEA BUSCAR

import os
import json
import subprocess
from datetime import datetime
from kivy.app import App
from kivy.metrics import dp
from kivy.uix.boxlayout import BoxLayout
from kivy.uix.gridlayout import GridLayout
from kivy.uix.image import Image
from kivy.uix.textinput import TextInput
from kivy.uix.button import Button
from kivy.uix.label import Label
from kivy.uix.popup import Popup
from kivy.uix.scrollview import ScrollView
from kivy.uix.filechooser import FileChooserListView
from kivy.graphics import Color, RoundedRectangle
import fnmatch

DATA_FILE = "inventario.json"
IMG_FOLDER = "fotos"

if not os.path.exists(IMG_FOLDER):
    os.makedirs(IMG_FOLDER)


class InventarioApp(App):

    def build(self):
        self.items = self.cargar_inventario()
        self.filtro_texto = ""

        root = BoxLayout(orientation="vertical", padding=dp(5), spacing=dp(5))

        # Barra de búsqueda
        filtro_layout = BoxLayout(orientation="horizontal", size_hint_y=None, height=dp(50), spacing=dp(5))
        self.filtro_input = TextInput(hint_text="🔍 Buscar (usa * para comodín)", multiline=False)
        btn_buscar = Button(text="Buscar", size_hint_x=None, width=dp(100), background_color=(0.3, 0.7, 1, 1))
        btn_buscar.bind(on_press=self.aplicar_filtro)
        filtro_layout.add_widget(self.filtro_input)
        filtro_layout.add_widget(btn_buscar)
        root.add_widget(filtro_layout)

        # Botón de agregar
        btn_add = Button(
            text="➕ Agregar producto",
            size_hint=(1, None),
            height=dp(55),
            background_color=(0.2, 0.6, 1, 1)
        )
        btn_add.bind(on_press=self.popup_agregar)
        root.add_widget(btn_add)

        # Lista de productos
        scroll = ScrollView(size_hint=(1, 1))
        self.lista = GridLayout(cols=1, spacing=dp(10), size_hint_y=None, padding=dp(10))
        self.lista.bind(minimum_height=self.lista.setter("height"))
        scroll.add_widget(self.lista)
        root.add_widget(scroll)

        self.actualizar_lista()
        return root

    # -----------------------------------
    # CARGAR Y GUARDAR INVENTARIO
    # -----------------------------------
    def cargar_inventario(self):
        if os.path.exists(DATA_FILE):
            with open(DATA_FILE, "r") as f:
                return json.load(f)
        return []

    def guardar_inventario(self):
        with open(DATA_FILE, "w") as f:
            json.dump(self.items, f, indent=4)

    # -----------------------------------
    # POPUP PARA AGREGAR PRODUCTO
    # -----------------------------------
    def popup_agregar(self, *args):
        layout = BoxLayout(orientation="vertical", padding=dp(10), spacing=dp(10))

        self.in_nombre = TextInput(hint_text="Nombre", size_hint_y=None, height=dp(45))
        self.in_desc = TextInput(hint_text="Descripción", size_hint_y=None, height=dp(100))

        btn_foto = Button(text="📸 Tomar foto (cámara nativa)", size_hint_y=None, height=dp(40))
        btn_foto.bind(on_press=self.tomar_foto)

        btn_galeria = Button(text="🖼 Seleccionar foto de galería", size_hint_y=None, height=dp(40))
        btn_galeria.bind(on_press=self.abrir_galeria)

        btn_guardar = Button(
            text="Guardar",
            background_color=(0, 1, 0, 1),
            size_hint_y=None,
            height=dp(45)
        )
        btn_guardar.bind(on_press=self.guardar_item)

        layout.add_widget(self.in_nombre)
        layout.add_widget(self.in_desc)
        layout.add_widget(btn_foto)
        layout.add_widget(btn_galeria)
        layout.add_widget(btn_guardar)

        self.ruta_foto = ""
        self.popup = Popup(title="Agregar producto", content=layout, size_hint=(0.9, 0.9))
        self.popup.open()

    # -----------------------------------
    # TOMAR FOTO SIN PLYER
    # -----------------------------------
    def tomar_foto(self, *args):
        nombre = f"foto_{datetime.now().strftime('%Y%m%d_%H%M%S')}.jpg"
        ruta = os.path.join(IMG_FOLDER, nombre)

        cmd = ["am", "start", "-a", "android.media.action.IMAGE_CAPTURE", "-o", ruta]

        try:
            subprocess.run(cmd)
            self.ruta_foto = ruta
            self.alerta("📸 La cámara se abrió.\nCuando tomes la foto, volverás aquí.")
        except Exception:
            self.alerta("❌ Error al abrir la cámara.\nActiva permisos para Pydroid.")

    # -----------------------------------
    # SELECCIONAR FOTO DESDE GALERÍA
    # -----------------------------------
    def abrir_galeria(self, *args):
        chooser = FileChooserListView(filters=[".jpg", ".png", "*.jpeg"])
        btn_sel = Button(text="Seleccionar", size_hint_y=None, height=dp(45))

        layout = BoxLayout(orientation="vertical")
        layout.add_widget(chooser)
        layout.add_widget(btn_sel)

        pop = Popup(title="Seleccionar imagen", content=layout, size_hint=(0.9, 0.9))

        def seleccionar(instance):
            if chooser.selection:
                self.ruta_foto = chooser.selection[0]
                pop.dismiss()

        btn_sel.bind(on_press=seleccionar)
        pop.open()

    # -----------------------------------
    # GUARDAR ITEM
    # -----------------------------------
    def guardar_item(self, *args):
        nombre = self.in_nombre.text.strip()
        desc = self.in_desc.text.strip()

        if nombre == "":
            self.alerta("⚠ El nombre no puede estar vacío.")
            return

        item = {"nombre": nombre, "desc": desc, "img": self.ruta_foto}
        self.items.append(item)
        self.guardar_inventario()
        self.actualizar_lista()
        self.popup.dismiss()

    # -----------------------------------
    # APLICAR FILTRO
    # -----------------------------------
    def aplicar_filtro(self, *args):
        self.filtro_texto = self.filtro_input.text.strip().lower()
        self.actualizar_lista()

    # -----------------------------------
    # ACTUALIZAR LISTA (con filtro y saltos de línea)
    # -----------------------------------
    def actualizar_lista(self):
        self.lista.clear_widgets()
        filtro = self.filtro_texto

        for index, item in enumerate(self.items):
            nombre = item["nombre"].lower()
            desc = item["desc"].lower()

            # Si hay filtro, se aplica con comodín (*)
            if filtro:
                patron = filtro.replace(" ", "*")
                coincide = (
                    fnmatch.fnmatch(nombre, f"{patron}") or
                    fnmatch.fnmatch(desc, f"{patron}")
                )
                if not coincide:
                    continue  # No coincide, no se muestra

            contenedor = BoxLayout(
                orientation="horizontal",
                size_hint_y=None,
                padding=dp(5),
                spacing=dp(10)
            )
            contenedor.height = dp(230)

            # Fondo gris suave
            with contenedor.canvas.before:
                Color(0.2, 0.2, 0.2, 0.3)
                contenedor.bg = RoundedRectangle(radius=[dp(10)], pos=contenedor.pos, size=contenedor.size)
            contenedor.bind(pos=lambda , _: setattr(contenedor.bg, "pos", contenedor.pos))
            contenedor.bind(size=lambda , _: setattr(contenedor.bg, "size", contenedor.size))

            # Imagen
            if item["img"] and os.path.exists(item["img"]):
                img = Image(source=item["img"], size_hint=(0.35, None), height=dp(200))
            else:
                img = Label(text="📦", font_size='35sp', size_hint=(0.35, None), height=dp(200))

            # Texto y botones
            info = BoxLayout(orientation="vertical", size_hint=(0.65, None), spacing=dp(10))
            info.height = dp(200)

            lbl_nombre = Label(
                text=f"[b]{item['nombre']}[/b]\n",
                markup=True,
                font_size='17sp',
                size_hint_y=None,
                halign="left",
                valign="middle"
            )
            lbl_nombre.bind(size=lambda s, _: setattr(s, "text_size", (s.width, None)))
            lbl_nombre.texture_update()
            lbl_nombre.height = lbl_nombre.texture_size[1] + dp(10)

            lbl_desc = Label(
                text=f"{item['desc']}\n\n",
                font_size='14sp',
                size_hint_y=None,
                halign="left",
                valign="top"
            )
            lbl_desc.bind(size=lambda s, _: setattr(s, "text_size", (s.width, None)))
            lbl_desc.texture_update()
            lbl_desc.height = lbl_desc.texture_size[1] + dp(10)

            # Botones editar / eliminar
            btn_layout = BoxLayout(size_hint_y=None, height=dp(35), spacing=dp(12), padding=(0, dp(5)))
            btn_editar = Button(text="✏️", size_hint=(None, None), width=dp(40), height=dp(35))
            btn_eliminar = Button(text="🗑️", size_hint=(None, None), width=dp(40), height=dp(35))
            btn_layout.add_widget(btn_editar)
            btn_layout.add_widget(btn_eliminar)

            btn_editar.bind(on_press=lambda inst, i=index: self.editar_item(i))
            btn_eliminar.bind(on_press=lambda inst, i=index: self.eliminar_item(i))

            info.add_widget(lbl_nombre)
            info.add_widget(lbl_desc)
            info.add_widget(btn_layout)

            contenedor.add_widget(img)
            contenedor.add_widget(info)
            self.lista.add_widget(contenedor)

    # -----------------------------------
    # EDITAR ITEM
    # -----------------------------------
    def editar_item(self, index):
        item = self.items[index]

        layout = BoxLayout(orientation="vertical", padding=dp(10), spacing=dp(10))
        in_nombre = TextInput(text=item["nombre"], size_hint_y=None, height=dp(45))
        in_desc = TextInput(text=item["desc"], size_hint_y=None, height=dp(100))
        btn_guardar = Button(
            text="Guardar cambios",
            size_hint_y=None,
            height=dp(45),
            background_color=(0, 1, 0, 1)
        )

        layout.add_widget(in_nombre)
        layout.add_widget(in_desc)
        layout.add_widget(btn_guardar)

        popup = Popup(title="Editar producto", content=layout, size_hint=(0.9, 0.9))

        def guardar_cambios(instance):
            item["nombre"] = in_nombre.text.strip()
            item["desc"] = in_desc.text.strip()
            self.guardar_inventario()
            self.actualizar_lista()
            popup.dismiss()

        btn_guardar.bind(on_press=guardar_cambios)
        popup.open()

    # -----------------------------------
    # ELIMINAR ITEM
    # -----------------------------------
    def eliminar_item(self, index):
        del self.items[index]
        self.guardar_inventario()
        self.actualizar_lista()

    # -----------------------------------
    # POPUP ALERTA
    # -----------------------------------
    def alerta(self, mensaje):
        pop = Popup(title="Información", content=Label(text=mensaje), size_hint=(0.8, 0.3))
        pop.open()


if _name_ == "_main_":
    InventarioApp().run()
#INVENTARIO

[
    {
        "nombre": "Filtro Suzuki K&N KN-138 el grande metalico",
        "desc": "Equivalencias: Suzuki V-Strom 650, GSX-R600, GSX-R750, SV650, DL650 y otros modelos Suzuki...",
        "img": ""
    },
    {
        "nombre": "Filtro Honda Hiflo HF-204",
        "desc": "Equivalencias: Honda CB500, CBR600RR, CB650F, NC700, NC750, CB500X, Hornet 600.",
        "img": ""
    },
    {
        "nombre": "Filtro Yamaha Hiflo HF-204",
        "desc": "Equivalencias: Yamaha MT-07, FZ-07, XSR700, T\u00e9n\u00e9r\u00e9 700, Tracer 700.",
        "img": ""
    },
    {
        "nombre": "Filtro Yamaha Hiflo HF-303",
        "desc": "Equivalencias: Yamaha R1, R6, MT-09 (modelos anteriores), FZ1, XJR1300.",
        "img": ""
    },
    {
        "nombre": "Carguero Bitrix (Filtro aceite mediano metalico",
        "desc": "Filtro de aire compatible con cargueros Bitrix tipo mototaxi.",
        "img": ""
    },
    {
        "nombre": "boxer",
        "desc": "Boxer (CT 100, BM 100, BM 150), Platino(100, 110, 125), Discover (100, 125 ST, 135, 150), Pulsar (135 LS, NS 125, NS 150, NS 160) y V15.\n\u200bYamaha: FZ16, FZ 2.0, Fazer 150, SZ-R 153 y XTZ 150.\n\u200bOtras: XCD 125, Wind 125, Caliber 115",
        "img": "/storage/emulated/0/DCIM/Camera/IMG_20251223_090834.jpg"
    },
    {
        "nombre": "ns 200",
        "desc": "\u200bBajaj: Pulsar NS 160, NS 125, AS 200, RS 200 y Dominar 250/400.\n\u200bKTM: Duke 125, 200, 250 y 390; RC 200 y 390.\n\u200bHusqvarna: Svartpilen 200/401 y Vitpilen 401",
        "img": ""
    },
    {
        "nombre": "ns200",
        "desc": "hahshdvdbbrbrbr cvvvcccvvvcc",
        "img": ""
    },
    {
        "nombre": "ns200",
        "desc": "",
        "img": "/storage/emulated/0/DCIM/Screenshots/Screenshot_2025-12-26-17-25-20-095_com.whatsapp-edit.jpg"
    },
    {
        "nombre": "nkd",
        "desc": "AKT: Modelos SL, SLR, NKDR, Evo (125/150), TT (125/150/180), TTR, TTX, CR5 180, XM 180 y XM 200",
        "img": ""
    },
    {
        "nombre": "dr",
        "desc": "dr650",
        "img": "fotos/foto_20260121_122351.jpg"
    },
    {
        "nombre": "r15",
        "desc": "FZ25, Fazer 250, XTZ 250 (Lander/T\u00e9n\u00e9r\u00e9), MT-15, Crypton 115 (FI), YBR 250 y XT 225.",
        "img": ""
    },
    {
        "nombre": "gn 125 rtr",
        "desc": "GS 125, GZ 150, DR 200, AN 125/150, Burgman 125.\n\u200bKeeway: RK-III, Superlight 200, TX 200.\n\u200bHaojue: TR 150, NK 150",
        "img": ""
    }
]
