# AppFilterOfOil
AppFilterOfOil

#CODIGO DE PYEDITER ESE
VOY A INTENTAR CON LAS IMAGENES LLENAR EL INVENTARIO DE TAL MANERA QUE SOLO SEA BUSCAR

import os
import json
import shutil
from kivy.app import App
from kivy.uix.boxlayout import BoxLayout
from kivy.uix.image import Image
from kivy.uix.label import Label
from kivy.uix.textinput import TextInput
from kivy.uix.button import Button
from kivy.uix.scrollview import ScrollView
from kivy.uix.popup import Popup
from kivy.uix.filechooser import FileChooserListView

DATA_FILE = "inventario.json"
IMG_FOLDER = "/storage/emulated/0/InventarioFotos"

if not os.path.exists(IMG_FOLDER):
    os.makedirs(IMG_FOLDER)


class InventarioApp(App):

    def build(self):
        self.data = self.cargar_datos()
        self.editando = None

        root = BoxLayout(orientation="vertical", padding=8, spacing=8)

        self.busqueda = TextInput(
            hint_text="Buscar por nombre, referencia o descripción",
            size_hint_y=None,
            height=45
        )
        self.busqueda.bind(text=self.actualizar_lista)
        root.add_widget(self.busqueda)

        self.lista = BoxLayout(
            orientation="vertical",
            spacing=12,
            size_hint_y=None
        )
        self.lista.bind(minimum_height=self.lista.setter("height"))

        scroll = ScrollView()
        scroll.add_widget(self.lista)
        root.add_widget(scroll)

        btn_agregar = Button(
            text="Agregar producto",
            size_hint_y=None,
            height=50
        )
        btn_agregar.bind(on_press=self.popup_agregar)
        root.add_widget(btn_agregar)

        self.actualizar_lista()
        return root

    # ---------- DATOS ----------

    def cargar_datos(self):
        if os.path.exists(DATA_FILE):
            with open(DATA_FILE, "r", encoding="utf-8") as f:
                return json.load(f)
        return []

    def guardar_datos(self):
        with open(DATA_FILE, "w", encoding="utf-8") as f:
            json.dump(self.data, f, indent=4, ensure_ascii=False)

    # ---------- LISTA ----------

    def actualizar_lista(self, *args):
        texto = self.busqueda.text.lower()
        self.lista.clear_widgets()

        for item in self.data:
            combinado = f"{item.get('nombre','')} {item.get('ref','')} {item.get('desc','')}".lower()
            if texto in combinado:
                self.lista.add_widget(self.tarjeta(item))

    def tarjeta(self, item):
        contenedor = BoxLayout(
            orientation="vertical",
            size_hint_y=None,
            padding=8,
            spacing=6
        )

        ruta_img = item.get("img", "")
        img = Image(
            source=ruta_img if os.path.exists(ruta_img) else "",
            size_hint_y=None,
            height=180,
            allow_stretch=True,
            keep_ratio=True
        )
        contenedor.add_widget(img)

        def crear_label(texto):
            lbl = Label(
                text=texto,
                markup=True,
                halign="left",
                valign="top",
                size_hint_y=None
            )
            lbl.bind(
                width=lambda s, w: setattr(s, "text_size", (w - 10, None)),
                texture_size=lambda s, ts: setattr(s, "height", ts[1] + 10)
            )
            return lbl

        contenedor.add_widget(crear_label(f"[b]Nombre:[/b] {item.get('nombre','')}"))
        contenedor.add_widget(crear_label(f"[b]Referencia:[/b] {item.get('ref','')}"))
        contenedor.add_widget(crear_label(f"[b]Descripción:[/b]\n{item.get('desc','')}"))

        botones = BoxLayout(size_hint_y=None, height=40, spacing=10)

        btn_editar = Button(text="Editar")
        btn_editar.bind(on_press=lambda x, i=item: self.popup_editar(i))

        btn_borrar = Button(text="Eliminar")
        btn_borrar.bind(on_press=lambda x, i=item: self.eliminar(i))

        botones.add_widget(btn_editar)
        botones.add_widget(btn_borrar)

        contenedor.add_widget(botones)

        contenedor.bind(
            minimum_height=lambda s, h: setattr(s, "height", h)
        )

        return contenedor

    # ---------- POPUPS ----------

    def popup_agregar(self, *_):
        self.editando = None
        self.popup_formulario("Agregar producto")

    def popup_editar(self, item):
        self.editando = item
        self.popup_formulario("Editar producto", item)

    def popup_formulario(self, titulo, item=None):
        layout = BoxLayout(orientation="vertical", spacing=8, padding=8)

        self.nombre = TextInput(hint_text="Nombre")
        self.ref = TextInput(hint_text="Referencia")
        self.desc = TextInput(hint_text="Descripción")

        self.ruta_img = ""

        if item:
            self.nombre.text = item.get("nombre", "")
            self.ref.text = item.get("ref", "")
            self.desc.text = item.get("desc", "")
            self.ruta_img = item.get("img", "")

        btn_img = Button(text="Seleccionar imagen", size_hint_y=None, height=45)
        btn_img.bind(on_press=self.selector_imagen)

        btn_guardar = Button(text="Guardar", size_hint_y=None, height=45)
        btn_guardar.bind(on_press=self.guardar_producto)

        layout.add_widget(self.nombre)
        layout.add_widget(self.ref)
        layout.add_widget(self.desc)
        layout.add_widget(btn_img)
        layout.add_widget(btn_guardar)

        self.popup = Popup(title=titulo, content=layout, size_hint=(0.9, 0.9))
        self.popup.open()

    # ---------- IMAGEN ----------

    def selector_imagen(self, *_):
        chooser = FileChooserListView(filters=["*.png", "*.jpg", "*.jpeg"])
        btn = Button(text="Usar imagen", size_hint_y=None, height=45)

        box = BoxLayout(orientation="vertical")
        box.add_widget(chooser)
        box.add_widget(btn)

        popup = Popup(title="Seleccionar imagen", content=box, size_hint=(0.9, 0.9))

        def seleccionar(*_):
            if chooser.selection:
                origen = chooser.selection[0]
                destino = os.path.join(IMG_FOLDER, os.path.basename(origen))
                shutil.copy(origen, destino)
                self.ruta_img = destino
                popup.dismiss()

        btn.bind(on_press=seleccionar)
        popup.open()

    # ---------- GUARDAR ----------

    def guardar_producto(self, *_):
        if not self.nombre.text or not self.ref.text:
            return

        if self.editando:
            self.editando["nombre"] = self.nombre.text
            self.editando["ref"] = self.ref.text
            self.editando["desc"] = self.desc.text
            if self.ruta_img:
                self.editando["img"] = self.ruta_img
        else:
            self.data.append({
                "nombre": self.nombre.text,
                "ref": self.ref.text,
                "desc": self.desc.text,
                "img": self.ruta_img
            })

        self.guardar_datos()
        self.popup.dismiss()
        self.actualizar_lista()

    # ---------- ELIMINAR ----------

    def eliminar(self, item):
        self.data.remove(item)
        self.guardar_datos()
        self.actualizar_lista()


if __name__ == "__main__":
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
